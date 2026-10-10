# Python Obfuscation (OBF) — Học từ số 0

> **Obf là gì?** Là viết lại code sao cho **máy vẫn chạy đúng** nhưng **người đọc rất khó hiểu**.
> **Deobf là gì?** Là quá trình ngược lại: từ code rối tìm lại code gốc / logic gốc.
>
> Tài liệu này dạy cả 2 chiều: **biết giấu → hiểu cách phá → biết giấu kỹ hơn**.

Học xong bạn sẽ làm được:
1. Giấu code bằng `exec` + `base64` / `zlib` / `bz2`.
2. Đọc và phá kiểu giấu đó bằng `print` và `hooking`.
3. Hiểu `marshal` + `bytecode` + `dis` là gì và vì sao nó khó hơn.
4. Dùng `AST` để tự viết tool obf string, số, `try/except`, `if` rác.
5. Hiểu tư duy `anti-hook` và combo `AST + marshal`.

### Cách đọc tài liệu này

- Đọc theo thứ tự Bài 0 → 4. Đừng nhảy.
- Mỗi bài có 4 phần: **Ý tưởng → Ví dụ chạy được → Cách phá → Bài tập**.
- Code trong bài đều copy chạy được trên Python 3.10+.

### Kiến thức cần có

- Biết `print`, biến, hàm, `if/try`, `import` cơ bản.
- Biết mở terminal chạy `python ten_file.py`.

### Mục lục

- [Bài 0: Bức tranh toàn cảnh](#bài-0-bức-tranh-toàn-cảnh)
- [Bài 1: Giấu code bằng exec và builtins](#bài-1-giấu-code-bằng-exec-và-builtins)
- [Bài 2: Bytecode, marshal và dis](#bài-2-bytecode-marshal-và-dis)
- [Bài 3: Dùng AST để viết tool obf](#bài-3-dùng-ast-để-viết-tool-obf)
- [Bài 4: Tư duy obf vs deobf nâng cao](#bài-4-tư-duy-obf-vs-deobf-nâng-cao)
- [Lỗi hay gặp (FAQ)](#lỗi-hay-gặp-faq)
- [Bảng thuật ngữ](#bảng-thuật-ngữ)
- [Cảm ơn](#cảm-ơn)

---

## Bài 0: Bức tranh toàn cảnh

Python chạy code qua 3 bước:

```text
source (.py)  --compile-->  bytecode  --VM thực thi-->  kết quả
     "print('hi')"        (dạng bytes khó đọc)      in ra hi
```

Hầu hết kỹ thuật obf trong tài liệu này chỉ là **chặn ở 1 trong 3 bước trên**:

| Bài | Chặn ở đâu | Ý tưởng 1 câu |
|-----|------------|----------------|
| Bài 1 | Giữ nguyên source (dạng chữ) | Bọc code trong `exec(...)` + mã hóa `base64/zlib` |
| Bài 2 | Đẩy xuống bytecode | Dùng `marshal` để code không còn là chữ nữa |
| Bài 3 | Viết lại source cho rối | Dùng `AST` biến `'hello'` thành `(lambda: 'hello')()` |
| Bài 4 | Chống người phá | Kết hợp tất cả + gài bẫy anti-hook |

Hai hàm quan trọng nhất cần nhớ:

- `exec(code_chuoi_hoac_bytes)`: nhận code, chạy luôn.
- `eval(...)`: giống `exec` nhưng chỉ dùng cho biểu thức, có trả về giá trị.

> Quy tắc vàng để deobf Bài 1: **chỗ nào gọi `exec` thì đó là chỗ lộ code gốc.**

---

## Bài 1: Giấu code bằng exec và builtins

### 1.1. Dạng giấu đơn giản nhất

```python
code = "print('hello world')"

# .encode() biến str -> bytes
print(code.encode("utf8"))  # b"print('hello world')"

# exec nhận cả str và bytes, đều chạy
exec(b"print('hello world')")  # in ra: hello world
```

Hiểu đơn giản: `exec` là “cái miệng” ăn code và chạy. Muốn giấu thì đừng để lộ `code`, mà bọc nó lại.

### 1.2. Thêm base64 vào

`base64` chỉ làm việc với `bytes`, không làm việc với `str`. Đây là lỗi 90% người mới gặp:

```python
import base64

code = "print('hello world')"

# SAI: base64.b64encode(code)
# TypeError: a bytes-like object is required, not 'str'

# ĐÚNG: encode trước rồi mới b64encode
data = base64.b64encode(code.encode("utf8"))
print(data)  # b'cHJpbnQoJ2hlbGxvIHdvcmxkJyk='
```

File obf lúc này chỉ còn 1 dòng:

```python
exec(__import__("base64").b64decode(b'cHJpbnQoJ2hlbGxvIHdvcmxkJyk='))
```

Vì sao dùng `__import__("base64")` thay vì `import base64`?
→ Để dồn hết thành 1 dòng, không lộ dòng `import` ở trên. Tác dụng che mắt là chính.

> Thứ tự cần nhớ: **lúc giấu làm gì trước thì lúc chạy giải ngược lại.**
> Giấu: `encode -> b64encode`. Chạy: `b64decode -> exec`.

### 1.3. Cách phá (deobf) loại này

Đừng đi giải `base64` bằng tay. Hãy nhìn logic:

```python
exec( GIẢI_MÃ( dữ_liệu ) )
```

`exec` là khâu cuối cùng. Muốn xem bên trong là gì, thay `exec` bằng `print`:

```python
# Gốc (obf)
exec(__import__("base64").b64decode(b'cHJpbnQoJ2hlbGxvIHdvcmxkJyk='))

# Để phá, sửa thành:
print(__import__("base64").b64decode(b'cHJpbnQoJ2hlbGxvIHdvcmxkJyk='))
# Kết quả: b"print('hello world')"
# .decode() thêm 1 cái nữa là ra chữ thường
```

Áp dụng tương tự cho `zlib`, `bz2` xếp chồng:

```python
import base64, bz2, zlib

code = "print('ngocuyen')"
# Giấu: b64 -> bz2 -> zlib
data = zlib.compress(bz2.compress(base64.b64encode(code.encode("utf8"))))

# Chạy: giải ngược lại zlib -> bz2 -> b64
exec(__import__("base64").b64decode(__import__("bz2").decompress(__import__("zlib").decompress(data))))
```

Muốn phá? Vẫn chỉ thay `exec` ngoài cùng bằng `print`. Không cần quan tâm có mấy lớp nén.

### 1.4. exec lồng nhau và kỹ thuật hooking

Kẻ giấu có thể bọc nhiều lớp `exec`:

```python
# Code sau 3 lớp bọc, chạy vẫn chỉ in hello world
exec('exec(\'exec("print(\\\'hello world\\\')")\')')
```

Nếu thay `exec = print` thủ công thì phải thay 3 lần, rất mệt. Cách pro hơn là **hooking**: gài camera theo dõi mọi lần gọi `exec`.

```python
hook_goc = exec  # giữ hàm gốc để tránh đệ quy vô tận

def hooking(code):
    print("Bắt được:", code)  # in ra trước khi chạy
    return hook_goc(code)     # rồi vẫn cho chạy bình thường

exec = hooking

exec('exec(\'exec("print(\\\'hello world\\\')")\')')
```

Kết quả bạn sẽ thấy cả quá trình bóc từng lớp:

```text
Bắt được: exec('exec("print(\'hello world\')")')
Bắt được: exec("print('hello world')")
Bắt được: print('hello world')
hello world
```

> Mở rộng: mọi hàm trong `builtins` (`print`, `input`, `int`, `chr`...) đều hook được theo cách này.

### Bài tập Bài 1

1. Tự viết 1 file obf `base64` cho `print('em da hieu')`, rồi tự phá nó bằng `print`.
2. Thử xếp 2 lớp `zlib + base64`, kiểm tra nếu giải sai thứ tự thì lỗi gì?
3. Viết hook cho `eval` thay vì `exec`.

**Tóm tắt Bài 1:** `exec` là chìa khóa. Giấu = mã hóa trước khi đưa vào `exec`. Phá = chặn ở `exec`.

---

## Bài 2: Bytecode, marshal và dis

### 2.1. Vì sao cần bytecode?

Ở Bài 1, code gốc vẫn là **chữ**. Hook là thấy ngay.

`marshal` biến code thành **bytecode** (dạng bytes đã biên dịch), không còn là chữ nữa:

```python
import marshal

# compile: chữ -> đối tượng code
# dumps: đối tượng code -> bytes
data = marshal.dumps(compile("print('hello')", "ten_file_ao.py", "exec"))
print(data[:60])  # b'c\x00\x00\x00\x00...' : một đống bytes khó đọc

# Muốn chạy: loads ngược lại rồi exec
exec(__import__("marshal").loads(data))  # in ra: hello
```

Lúc này nếu bạn thay `exec = print` thì chỉ nhận được:

```text
<code object <module> at 0x...>
```

→ Không còn chữ để đọc. Buộc phải dùng công cụ khác: `dis`.

### 2.2. Dùng dis để nhìn vào bên trong

`dis` là kính hiển vi xem bytecode đang làm gì:

```python
import marshal, dis

data = marshal.dumps(compile("print('hello')", "ten_file_ao.py", "exec"))
code_obj = marshal.loads(data)

# Thay exec = dis.dis để xem thay vì chạy
__import__("dis").dis(code_obj)
```

Kết quả rút gọn (Python 3.11+):

```text
LOAD_NAME   print
LOAD_CONST  'hello'
CALL        1
POP_TOP
RETURN_VALUE
```

Bạn chỉ cần hiểu 3 lệnh chính:

| Lệnh | Nghĩa đời thường |
|------|------------------|
| `LOAD_NAME print` | Lấy hàm `print` ra |
| `LOAD_CONST 'hello'` | Lấy giá trị `'hello'` ra |
| `CALL` | Gọi hàm với giá trị trên (`print('hello')`) |

> Lưu ý quan trọng: tên opcode thay đổi theo phiên bản Python (3.10 có `PRECALL`, 3.12+ bỏ `PRECALL`, 3.14 lại khác). Đừng học thuộc số opcode, hãy học cách đọc `dis` trên máy của bạn.

### 2.3. Công cụ có sẵn và giới hạn

- `pycdc`, `uncompyle6` / `decompyle3`: dịch `.pyc` / bytecode về `.py`.
- Điểm mạnh: nhanh. Điểm yếu: hay lỗi với Python mới, thiếu opcode, code obf nặng thì dịch sai.

Vì vậy người viết obf thích dùng `marshal`: vừa dễ, vừa làm công cụ tự động hỏng.

### 2.4. Mini VM giải thích cho vui (không bắt buộc)

Ý tưởng của đoạn “mini decompile” trong bản cũ:

```text
lặp từng opcode trong code_obj.co_code:
  gặp LOAD_CONST -> đẩy giá trị vào ngăn xếp (stack)
  gặp LOAD_NAME  -> đẩy hàm vào ngăn xếp
  gặp CALL       -> lôi hàm + đối số ra, gọi thật, ghi lại thành chữ
```

Đó chính là cách Python VM chạy code. Viết lại VM mini giúp bạn hiểu, nhưng tốn công (tác giả gốc mất 5 tiếng chỉ cho `print`). Người mới **chỉ cần hiểu ý tưởng**, chưa cần viết được.

### Bài tập Bài 2

1. `compile` thử `a = 1 + 2`, rồi `dis` nó. Tìm xem `BINARY_OP` nằm ở đâu?
2. Thử `marshal` 1 file có `def hello():`, `dis` ra và đếm có mấy `MAKE_FUNCTION`?
3. Cài `pycdc` thử dịch 1 file `.pyc` của bạn, ghi lại lỗi gặp phải (nếu có).

**Tóm tắt Bài 2:** `marshal` giấu chữ thành bytes. `print/hook` vô dụng. Phải `dis` hoặc dùng tool dịch ngược.

---

## Bài 3: Dùng AST để viết tool obf

### 3.1. AST là gì? (hiểu trong 2 phút)

`AST` (Abstract Syntax Tree) là **cây cú pháp**: Python đọc code chữ, vẽ thành cây, rồi mới sinh bytecode.

```python
import ast

code = "print(1000)"
tree = ast.parse(code)
print(ast.dump(tree, indent=2))
```

Ra:

```text
Module(
  body=[
    Expr(
      value=Call(
        func=Name(id='print'),
        args=[Constant(value=1000)]))])
```

Đọc từ trong ra: `Constant(1000)` là quả, `Name(print)` là dao, `Call` là hành động bổ. Hiểu cây thì ta có thể **đi bộ qua cây (`ast.walk`), thay quả (`Constant`) bằng quả khác**, rồi in lại thành code (`ast.unparse`).

Quy trình tool obf AST:

```text
parse (chữ -> cây) -> walk + thay node -> unparse (cây -> chữ mới rối hơn)
```

### 3.2. Chiêu 1: Giấu string và số bằng lambda

Vì sao:

```python
print("hello")
# tương đương 100% với:
print((lambda: "hello")())
```

`(lambda: "hello")()` nghĩa là “định nghĩa hàm ẩn danh trả về hello rồi gọi luôn”. Máy chạy như nhau, mắt người rối hơn.

Tool sẽ tự thay mọi `Constant` chuỗi/số:

```python
# Trước
print(1000)
print("ngocuyen")

# Sau 1 vòng obf
print(int('1000'))
print((lambda: 'ngocuyen')())
```

Lặp 5 vòng thì lambda lồng 5 lớp. Càng lặp càng rối.

Code lõi (đã rút gọn, chạy được):

```python
import ast

def obfstr(v: str):
    # v = hello -> (lambda: "hello")()
    return f'(lambda: "{v}")()'

def obfint(v: int):
    # trick: biến số thành int("1000") để tiếp tục obf chuỗi "1000" bên trong
    return f'int("{v}")'

def obfuscate(tree: ast.AST):
    for node in ast.walk(tree):
        for field, value in ast.iter_fields(node):
            if isinstance(value, list):
                new_list = []
                for item in value:
                    if isinstance(item, ast.Constant) and isinstance(item.value, str):
                        new_list.append(ast.parse(obfstr(item.value)).body[0].value)
                    elif isinstance(item, ast.Constant) and type(item.value) is int:
                        # dùng type() is int để loại True/False (vì bool là con của int)
                        new_list.append(ast.parse(obfint(item.value)).body[0].value)
                    elif isinstance(item, ast.AST):
                        new_list.append(item)
                # nếu list chứa True/False thì giữ nguyên để tránh hỏng logic
                if any(isinstance(e, ast.Constant) and isinstance(e.value, bool) for e in value):
                    continue
                setattr(node, field, new_list)
            elif isinstance(value, ast.Constant) and isinstance(value.value, str):
                setattr(node, field, ast.parse(obfstr(value.value)).body[0].value)
            elif isinstance(value, ast.Constant) and type(value.value) is int:
                setattr(node, field, ast.parse(obfint(value.value)).body[0].value)

def obf(src: str):
    tree = ast.parse(src)
    obfuscate(tree)
    return ast.unparse(tree)
```

> 3 bẫy người mới hay dính (bản cũ từng dính):
> 1. `True/False` cũng là `int` trong Python (`isinstance(True, int) == True`). Phải dùng `type(x) is int` hoặc skip `bool` riêng, nếu không `True` thành `int('True')` là sai.
> 2. Chuỗi rỗng `""` và ký tự `\n`, `\r`, dấu nháy: phải dùng `repr(c)` thay vì `f'"{c}"'` để không vỡ code.
> 3. `f"{a}"` là node `JoinedStr`, không phải `Constant`. Muốn obf được phải đổi về `"{}".format(a)` trước (xem 3.3).

### 3.3. Xử lý f-string

```python
hello = 5
print(f"{hello}")  # AST: JoinedStr, không bắt bằng Constant được
```

Cách xử lý: viết hàm đổi `JoinedStr` → `.format`:

```python
def fm(node: ast.JoinedStr) -> ast.Call:
    return ast.Call(
        func=ast.Attribute(
            value=ast.Constant(value='{}' * len(node.values)),
            attr="format",
            ctx=ast.Load()
        ),
        args=[v.value if isinstance(v, ast.FormattedValue) else v for v in node.values],
        keywords=[]
    )
# f"{a} tuoi {b}" -> "{} tuoi {}".format(a, b)
```

Thêm 2 nhánh `elif isinstance(v, ast.JoinedStr)` vào `obfuscate()` là xong.

### 3.4. Làm cho lambda “có hồn” hơn

Lambda chay `(lambda: 'hi')()` nhìn vẫn dễ đoán. Thêm 2 gia vị:

1. Tên tham số rác:

```python
# Trước
(lambda: 'hi')()
# Sau (tên Hán tự hoặc _0x1234)
(lambda 駹穯: 'hi')('駹穯')
(lambda _0x3327: 'hi')('_0x3327')
```

2. Tách chuỗi:

```python
# 'hi' -> 'h' + 'i', mỗi mảnh lại bọc lambda tiếp
(lambda _0x1: (lambda _0x1: 'h')('_0x1') + (lambda _0x1: 'i')('_0x1'))('_0x1')
```

Cài đặt chỉ cần đổi `obfstr`:

```python
import random
randomchar = "_0x" + "".join(random.choices("123456789", k=4))

def obfstr(v: str):
    if v == "":
        return "''"
    tach = "+".join(repr(c) for c in v)  # repr để giữ \n, ', "
    return f'(lambda {randomchar}: {tach})("{randomchar}")'
```

> Mẹo: `randomchar` nên sinh mới mỗi vòng lặp obf, nếu không mọi lambda trùng tên nhìn dễ đoán.

### 3.5. Chiêu 2: Bọc try/except rác

Ý tưởng: mỗi câu lệnh gốc đem bọc trong 4 lớp:

```python
# Gốc
print("hello world")

# Sau khi bọc 1 lớp
try:
    raise MemoryError('Ngocuyencoder')
except MemoryError:
    print("hello world")
```

Máy chạy: cố tình `raise` rồi bắt ngay, cuối cùng vẫn chạy `print`. Mắt người: toàn `try` lồng nhau.

Code tạo node (dùng `ast.Try`, `ast.Raise`):

```python
def trycatch(body, loop: int):
    out = []
    for stmt in body:
        j = stmt
        for _ in range(loop):
            j = ast.Try(
                body=[
                    ast.Raise(
                        exc=ast.Call(
                            func=ast.Name(id='MemoryError', ctx=ast.Load()),
                            args=[ast.Constant(value="Ngocuyencoder")],
                            keywords=[]
                        ),
                        cause=None
                    )
                ],
                handlers=[ast.ExceptHandler(
                    type=ast.Name(id='MemoryError', ctx=ast.Load()),
                    body=[j]
                )],
                orelse=[], finalbody=[]
            )
        out.append(j)
    return out
```

### 3.6. Chiêu 3: Nhét if rác

Câu lệnh không bao giờ đúng, chỉ để làm dài code:

```python
if False is True:
    pass
else:
    pass
```

Tạo bằng AST:

```python
def random_if_else():
    return ast.If(
        test=ast.Compare(
            left=ast.Constant(value=False),
            ops=[ast.Is()],
            comparators=[ast.Constant(value=True)]
        ),
        body=[ast.Pass()],
        orelse=[ast.Pass()]
    )
```

Nhét nó vào đầu mỗi `try` ở mục 3.5 (`body=[random_if_else(), raise...]`).

### 3.7. Gộp lại thành tool hoàn chỉnh

Xem file `astobf.py` trong repo này — đó chính là bản gộp 3 chiêu trên. Cách dùng:

```python
code = 'print("hello world")'
loop_try = 2   # số lớp try lồng nhau. Cảnh báo: >8 dễ lỗi too many statically nested blocks
loop_str = 3   # số lần lặp obf string

for _ in range(loop_str):
    code = obf(code, loop_try)
print(code)
```

> Đừng tham `loop` cao. `loop_try=4` + `loop_str=3` đã đủ rối, cao hơn dễ sinh file hàng nghìn dòng, Python báo `SyntaxError`.

### Bài tập Bài 3

1. Chạy `ast.dump` với `x = 1 + n`, chỉ ra đâu là `BinOp`, `Add`, `Name`.
2. Đổi `obfint` để số `1000` thành `int('1'+'0'+'0'+'0')`. Chạy thử còn đúng không?
3. Tự thêm chiêu mới: đổi tên biến `hello` thành `_0x...` bằng AST (`ast.Name`).

**Tóm tắt Bài 3:** AST cho phép viết lại code hàng loạt mà không sợ sai logic, miễn là thay đúng node.

---

## Bài 4: Tư duy obf vs deobf nâng cao

### 4.1. Hook và anti-hook

Vì Bài 1 quá dễ bị hook, người viết obf nghĩ ra **anti-hook**: kiểm tra xem `exec` có bị thay không.

```python
# exec gốc có kiểu builtin_function_or_method
print(type(exec))  # <class 'builtin_function_or_method'>

# Sau khi hook, kiểu đổi thành function
c = exec
def h(x):
    return c(x)
exec = h
print(type(exec))  # <class 'function'>
```

Anti đơn giản:

```python
if str(type(exec)) != "<class 'builtin_function_or_method'>":
    raise MemoryError("Phat hien hook")
```

Nhưng anti này cũng dễ bypass: hook luôn cả `type` và `str`:

```python
exec = h
def type(x):
    return "<class 'builtin_function_or_method'>"
```

→ Bài học: **không có anti tuyệt đối nếu đối thủ vẫn được phép chạy code của bạn.** Anti chỉ làm chậm, làm nản.

Hai kiểu trừng phạt hay dùng khi phát hiện hook:

```python
# 1. Ném lỗi
if bi_hook: raise MemoryError("Doi hook a")

# 2. Treo máy / tràn RAM (rất khó chịu, cân nhắc khi dùng)
if bi_hook: x = [[0] * 10**9]
```

### 4.2. Vì sao không nên lộ exec/eval?

Ví dụ giấu `exec` bằng đảo chuỗi vẫn chết:

```python
__builtins__.__dict__[''.join(["c","e","x","e"][::-1])](")'olleh'(tnirp"[::-1])
# thực chất là exec("print('hello')")
```

Chỉ cần hook đúng hàm `exec` gốc dưới lớp vỏ đó là lấy được code:

```python
goc = exec
def h(x):
    print(x)  # in ra print('hello')
    return goc(x)
exec = h
__import__("builtins").exec = h
```

→ Quy tắc: **còn gọi `exec/eval` dạng chữ thì còn hook được.** Muốn hết hook thì phải xuống bytecode (Bài 2).

### 4.3. Combo mạnh nhất cho người mới: AST + marshal

1. Dùng AST (Bài 3) làm rối source: lambda, try, if rác.
2. `compile + marshal.dumps` đống rối đó thành bytes.
3. File phát hành chỉ còn:

```python
exec(__import__("marshal").loads(b'c\x00... rất dài ...'))
```

Đối thủ lúc này phải:

- `dis` cả trăm dòng toàn `MAKE_FUNCTION`, `BINARY_OP`, `RAISE_VARARGS`...
- Lần theo từng `lambda` nhỏ (`'h'`, `'e'`, `'l'`...) rồi ráp lại.
- Vừa chống anti-hook, vừa dịch try/except.

Rất nản → đa số bỏ cuộc. Đó là lý do ở Việt Nam combo này phổ biến (không tính đóng gói exe/elf).

Minh họa rút gọn khi `dis` combo trên (đã cắt bớt, máy bạn có thể hiện opcode hơi khác theo version):

```text
LOAD_CONST False | LOAD_CONST True | IS_OP      # if False is True
PUSH_NULL | LOAD_NAME MemoryError | CALL | RAISE # raise MemoryError
...
LOAD_CONST <code object <lambda>> | MAKE_FUNCTION # (lambda: 'h')
BINARY_OP +                                     # 'h' + 'e' + ...
CALL print
```

Mỗi chữ `hello world` bị chẻ thành 11 lambda, mỗi lambda là 1 code object riêng.

### Bài tập Bài 4

1. Viết anti-hook cho `print` thay vì `exec`.
2. Tự bypass anti ở mục 4.1 bằng cách hook `str`.
3. Làm 1 file demo full pipeline: `source -> AST obf (1 vòng) -> marshal -> dis thử`.

**Tóm tắt Bài 4:** Obf giỏi không phải là mã hóa mạnh nhất, mà là **làm đối thủ tốn nhiều thời gian nhất với công sức ít nhất của mình.**

---

## Lỗi hay gặp (FAQ)

| Lỗi / Hiện tượng | Nguyên nhân | Cách sửa |
|------------------|-------------|----------|
| `TypeError: a bytes-like object is required` | `b64encode` với `str` | `.encode("utf8")` trước |
| Hook xong đệ quy vô tận | `exec = hooking` mà trong `hooking` lại gọi `exec` | Giữ `hook_goc = exec` trước, trong hàm gọi `hook_goc` |
| `True` biến thành `int('True')` | `bool` là con của `int` | Dùng `type(x) is int`, skip `bool` |
| `f"..."` obf không hết | `f-string` là `JoinedStr` | Đổi về `.format` bằng hàm `fm()` |
| `SyntaxError: too many statically nested blocks` | `loop` try quá cao | Giảm `loop_try` xuống 2–4 |
| `dis` trên máy khác bản cũ | Opcode đổi theo version Python | Đọc `dis` trên đúng version đang chạy, đừng học thuộc số |
| `ast.Str` báo deprecated | Python 3.8+ dùng `Constant` | Dùng `ast.Constant(value=...)` |
| File obf chạy chậm, phình to | Lặp string nhiều vòng | Chỉ 2–3 vòng, kiểm tra dung lượng sau mỗi vòng |

---

## Bảng thuật ngữ

| Từ | Hiểu nhanh |
|----|------------|
| `builtins` | Hộp đồ nghề mặc định của Python (`print`, `exec`, `int`...) |
| `exec / eval` | Hàm ăn code chữ rồi chạy |
| `hook` | Thay hàm gốc bằng hàm của mình để nghe lén / ghi log |
| `marshal` | Đóng gói code đã compile thành bytes |
| `bytecode` | Dạng trung gian giữa chữ và máy chạy |
| `dis` | Công cụ xem bytecode đang làm gì |
| `AST` | Cây cú pháp vẽ từ code chữ |
| `parse / unparse` | Chữ → cây / cây → chữ |
| `lambda` | Hàm ẩn danh 1 dòng, hay dùng để bọc chuỗi |
| `anti-hook` | Bẫy phát hiện ai đó đang hook |

---

## Cảm ơn

- Nguyễn Văn Khánh: hỗ trợ test obf (Dec_Kramer, Dump_Marshal).
- Nguyễn Song Hải Anh: định hướng bytecode và thế nào là obf mạnh.
- Nguyễn Minh: cộng tác vượt khó.

> Tài liệu được viết lại cho dễ học từ bản gốc. Nếu thấy chỗ nào vẫn khó hiểu, hãy mở issue với đoạn code + version Python bạn đang dùng.

