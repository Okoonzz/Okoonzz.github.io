+++
author = true
title = 'Rev sth'
date = 2024-05-10T10:16:18+07:00
image = 'img1.png'
+++


# HITCON_2023

*Tải file [tại đây.](https://github.com/Okoonzz/07.-Rev/blob/File/Hitcon2023.zip)*

## LessEQualmore

*Bài này dựa trên bài tham khảo của brosu (người đã hoàn thành trong giải)*

Trước khi tìm hiểu cách giải bài này ta sẽ cùng tham khảo qua bài viết [tại đây.](https://towardsdatascience.com/essential-math-for-data-science-matrices-as-linear-transformations-6d1d0e628131)

Bài này đầu tiên khi debug thật sự cũng chả thấy gì, ngoài thấy chỉ load nội dung ở file `chal.txt` vào (nhưng có lẽ sẽ rất khó thấy quy luật bởi có lúc thì nó sẽ nhảy tới 3 có lúc thì nhảy tới 6) và sau đó nhập vào để check flag. Sau khi một hồi ngồi decompile lại toàn bộ nội dung của chương trình, thì sẽ thấy được đây chính là một bài `vm` dùng để check flag.

Không hiểu sao lần này IDA trả về mã giả khá ngu. Đây chính là nội dung mà IDA trả về:

![](https://hackmd.io/_uploads/HJa3t8x1p.png)

Nếu để ý kỹ hơn thì ta sẽ so sánh với ASM như sau:

![](https://hackmd.io/_uploads/H13CtLxJ6.png)

Ta thấy được ở ASM chỉ với hai biến được sử dụng đó là `v1` và `re` nhưng ở mã giả trả về lại tới 3.  ¯\_(ツ)_/¯

Vì đoạn mã ASM này cũng khá ngắn nên mình đã quyết định viết lại nó và đây là toàn bộ đoạn mã của chương trình:

```python
mem : list[int] = []

def is_negative(a: int) -> bool:
    return a < 0


def getchar() -> str:
    ip = input("Enter char: ")
    if len(ip) == 0:
        return '\n'
    else:
        return ip[0]


def get_bignum() -> int:
    v1 : str
    ret : int
    v1 = getchar()
    if v1 == '%':
        v1 = getchar()
        if v1 == 'x':
            ret = 0
            v1 = getchar()
            while v1 != '\n':
                ret *= 16
                if ord(v1) <= 64 or ord(v1) > 70:
                    if ord(v1)  <= 96 or ord(v1) > 102:
                        ret += ord(v1) - 48
                    else:
                        ret += ord(v1) - 87
                else:
                    ret += ord(v1) - 55
                v1 = getchar()
        else:
            if ord(v1) <= 0x78:
                if v1 == '%':
                    ret = ord(v1)
                    return -ret
                if ord(v1) >= 0x25 and ord(v1) - 48 <= 9:
                    ret = ord(v1) - 48
                    v1  = getchar()
                    while v1 != '\n':
                        ret = 10 * ret + ord(v1) - 48
                        v1 = getchar()
                    return -ret
                else:
                    return - ord(v1)
            else:
                return -ord(v1)
    else:
        return -ord(v1)

def bignum_sub(a1: int, a2: int) -> int:
    return a2 - a1

def op1(a1: int, a2: int):
    bignum : int
    tmp1 = is_negative(a1)
    tmp2 = is_negative(a2)
    if tmp1:
        bignum = get_bignum()
    else:
        bignum = mem[a1]
    if tmp2:
        print(chr(bignum), end='')
    else:
        mem[a2] = bignum_sub(bignum, mem[a2])

def runProg():
     ins = 0
     while ins >= 0:
          op1(mem[ins], mem[ins +1])
          if mem[mem[ins +1]] <= 0:
               ins = mem[ins + 2]
          else:
               ins += 3



def readProg(filename : str):
	with open(filename, "r") as file:
		for line in file.readlines():
			mem.extend([int(a) for a in line.strip("\n ").split(" ")])


def main():
    readProg("chal.txt")
    runProg()

if __name__ == "__main__":
	main()
```
> Toàn bộ hàm trên mình đã decompile bằng IDA chỉ có readProg là tham khảo thêm của brosu (vì dài quá nên mình lười :)) )

Có thể kiểm tra double check với debug. Qua đó ta sẽ thấy rõ hơn đây chính là `vm` và nó sẽ check từng chữ. Nhưng check bằng cách nào ??? Ta sẽ tiến hành đào sâu vào nó.

Đầu tiên với `getchar` ta sẽ thử đổi lại lưu toàn bộ `input` vào thay vì chỉ lưu từng ký tự:

```python
strBuf : str
def getchar() -> str:
    global strBuf
    if len(strBuf) == 0:
        ip = '\n'
    else:
        ip = strBuf[0]
        strBuf = strBuf[1:]
    return ip
```
>`VM` có đặc tính đó chính là chạy những `ins` để setup và sau đó sẽ thao tác với đầu vào và sau cùng là so sánh trên những `ins` đó, đặc biệt mỗi lần chạy và setup như thế thì nó chỉ thao tác với đúng một ký tự trong flag, và nếu chỉ 1 chữ sai thì sẽ trả về toàn bộ sai, đôi khi cũng có những chương trình không cần detect 1 chữ sai thì sẽ trả về sai mà nó vẫn sẽ lưu lại đó và thao tác tiếp những ký tự tiếp theo nhưng khi thao tác ký tự mới thì nó sẽ load lại những `ins`. 

Bây giờ, ta sẽ kiểm tra xem nó đang thực sự làm gì với đầu vào, ta sẽ đặt `cnt` tại hàm `runProg()` để đếm xem số lần chạy những `ins` cho những lần ký tự được nhập vào. Có vẻ như chương trình không phải là loại nếu gặp 1 ký tự sai thì thoát ngay, nên mình sẽ tận dụng điều này để nhập mỗi lần một tăng dần cho chương trình. Nó cũng giống như cách mình brute, ví dụ nó sẽ dựa vào những chữ đúng phía trước rồi sẽ brute ký tự tiếp theo, còn đối với bài này thì tương tự như thế nhưng bây giờ sẽ ví dụ như mình vẫn chưa biết được các ký tự đầu tiên là gì nên mình sẽ nhập lần lượt `a, aa, aaa,...` :

```python
inList : list[int] = []
def plotgrap(inList: list[int]):
    plt.scatter([i for i in range(len(inList))], inList)
    plt.show()

def main():
    global strBuf
    global inList
    for i in range(0,100):
        strBuf = "a"*i
        readProg("chal.txt")
        inList.append(runProg())
    print(strBuf)
    plotgrap(inList)
```
![](https://hackmd.io/_uploads/B17aUsg1a.png)

Ta sẽ nhận được một đồ thị như hình trên, với trục hoành là số ký tự nhập vào, còn trục tung là số `ins` sau tất cả lần chạy với những ký tự nhập vào.

Thấy được rằng 7 chunk đầu tiên mỗi chunk gồm 8 ký tự được phân chia khá rõ ràng. 

Vì đã biết được đầu flag có dạng `hitcon{` nhưng dạng này chỉ có 7 ký tự, nhưng trên hình mình đã đoán nó phải 8 ký tự nên sẽ brute ký tự còn lại. Cách brute được thực hiện như sau:

```python
inList : list[int] = []
charList = []
def plotgrap(charList, inList: list[int]):
    plt.scatter(charList, inList)
    plt.show()

def main():
    global strBuf
    global inList
    global charList
    for ch in printable:
        charList.append(ch)
        strBuf = "hitcon{" + ch
        readProg("chal.txt")
        inList.append(runProg())
    
    print(strBuf)
    plotgrap(charList, inList)
```

![](https://hackmd.io/_uploads/B1_3poxka.png)

Để ý thấy được số lượng `ins` tăng đột biến ở `r` nên khả năng cao 8 bytes đầu của flag là `hitcon{r`

Ta sẽ thử lại một lần nữa và đây chính là kết quả:

![](https://hackmd.io/_uploads/SyQPCoeJT.png)

Thấy được lúc đầu 7 chunk, lúc sau chỉ còn 6 chunk vậy giả thiết ban đầu đã đúng. Chương trình được chia làm 8 chunk mỗi chunk có 8 bytes để so sánh. (vì rất có thể chunk cuối cùng không có sự phân chia vì nó đã sai hoàn toàn nên nó sẽ dính liền với những ký tự rác phía sau)

Vậy câu hỏi bây giờ là có brute được không ?? Có thể là không bởi brute một lần 8 byte mà tới 256 ký tự, =)))

Vì đã viết ra được chương trình, nên ta sẽ xem trong số các `ins` load vào để thực hiện `vm` đó thì đâu là nơi chúng lưu những byte đầu tiên của input. Vì như đã nói ban đầu `ins` không chỉ có thao tác và kiểm tra mà nó còn thêm cả setup nên không đoán được chúng sẽ lưu ở đâu.

Vì mem chỉ thao tác ở hàm `runProg()` nên ta sẽ đào sâu việc tìm mem ở đây:

```python
def runProg():
    cnt = 0
    ins = 0
    while ins >= 0:
        cnt += 1
        op1(mem[ins], mem[ins +1])
        if mem[mem[ins +1]] <= 0:
               ins = mem[ins + 2]
        else:
               ins += 3
        if cnt % 1500 == 0:
            for i in range(len(mem)):
                if mem[i:i+8] == [104, 105, 116, 99, 111, 110, 123, 114]:
                    print(f"idx: {i}")
    return cnt
# output: 16
```
Như vậy đã xác định được vị trí đầu tiên lưu đầu vào nhập vào chính là `mem[16]`

Có được vị trí đầu vào tiếp đến, ta sẽ xem xem sau khi chạy xong `vm` thì đầu ra của ta sẽ nhận được những gì. Vì như đã nói ban đầu bài này phải nhập hết rồi nó sẽ kiểm tra từng từ nên mình sẽ giả dụ như đã nhập đủ flag và xem kết quả được in ra:

```python
def main():
    global strBuf
    global inList
    global charList
    strBuf = 'hitcon{r' + 'aaaaaaaa' + 'aaaaaaaa' + 'aaaaaaaa' + 'aaaaaaaa' + 'aaaaaaaa' + 'aaaaaaaa' + 'aaaaaaaa'
    tmp = strBuf
    readProg("chal.txt")
    runProg()
    for i in range(16, len(tmp),8):
        print(mem[i:i+8], end=' ')
# output: [16774200, 1411, 16776275, 3646, 1532, 6451, 2510, 16777141] [16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0] [16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0] [16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0] [16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0] [16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0]
```
Tương tự như lúc debug, ta sẽ thử thay đổi 1 ký tự trong chuỗi `aaaaaaaa` thành `aaaaaaab` để xem nó thành gì. Và đây là kết quả:

```
[16774200, 1411, 16776275, 3646, 1532, 6451, 2510, 16777141]
[16774493, 1161, 16776349, 3305, 1265, 5634, 2341, 15]
[16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0]
[16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0]
[16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0]
[16774500, 1164, 16776343, 3298, 1261, 5626, 2328, 0]
```
Thấy rằng từ hàng thứ 2 và những hàng sau có sự thay đổi khi thay giá trị. Cụ thể nó đã thay đổi một lượng `-7	-3	6	7	4	8	13	15`, những dãy dưới tương tự vì cùng `aaaaaaaa` không thay đổi. Làm tương tự cho các index còn lại của `aaaaaaaa` ví dụ tiếp đến thành `aaaaaaba` ,...
ta sẽ thu được bảng như sau:

```
-7	-3	6	7	4	8	13	15
-2	11	-2	2	9	20	-6	-11
-13	-10	-5	17	-8	1	22	15
4 	6 	4 	-6 	6 	5 	-8 	-5
-4 	5 	-3 	5 	3 	13 	1 	-6
3 	2 	-3 	-3 	-1 	-2 	-6 	-8
-2 	3 	-4 	3 	2 	7 	-1 	-5 
-7 	-2 	-2 	9 	-2 	6 	9 	5 
```
Vậy tổng quan của chương trình đó là `input -> 8 chunk -> biến đổi qua những hằng số nhất định -> so sánh với enc`

Như bài viết đã tham khảo ban đầu 8 chunk chúng biến đổi qua những hằng số và có kết quả mới, thì cũng tương tự như việc nhân ma trận AX với A là ma trận chứa -7,-3,... còn X là ma trận chứa 8 chunk. Việc cuối cùng ta chỉ cần giải phương trình AX=B là hoàn tất

Đây là script giải của bài này:
```python
from sage.all import*

A = [[-7, -2, -2, 9, -2, 6, 9, 5], [-2, 3, -4, 3, 2, 7, -1, -5], [3, 2, -3, -3, -1, -2, -6, -8], [-4, 5, -3, 5, 3, 13, 1, -6], [4, 6, 4, -6, 6, 5, -8, -5], [-13, -10, -5, 17, -8, 1, 22, 15], [-2, 11, -2, 2, 9, 20, -6, -11], [-7, -3, 6, 7, 4, 8, 13, 15]]
x = Matrix(Zmod(2**24),A)
y = x.inverse()
# print(y)

chal = [[16774200, 1411, 16776275, 3646, 1532, 6451, 2510, 16777141], [16775256, 2061, 16776706, 2260, 2107, 6124, 878, 16776140], [16775299, 1374, 16776956, 2212, 1577, 4993, 1351, 16777040], [16774665, 1498, 16776379, 3062, 1593, 5966, 1924, 16776815], [16774318, 851, 16775763, 3663, 711, 5193, 2591, 16777069], [16774005, 1189, 16776283, 3892, 1372, 6362, 2910, 307], [16775169, 1031, 16776798, 2426, 1171, 4570, 1728, 33], [16775201, 819, 16776898, 2370, 1132, 4255, 1900, 347]]

for tar in chal:
    target = vector(tar)
    decr = vector(target*y)
    print(decr)
```

## CrazyArcade 

Đây là một bài chạy với file .exe. Ta sẽ mở phân tích nó bằng IDA

![](https://hackmd.io/_uploads/rJF400E16.png)

Đầu tiên với hàm main, hầu như đã có tất cả symbol nên việc nhìn thấy công dụng của chúng khá dễ dàng, chỉ phát hiện duy nhất một hàm này:

![](https://hackmd.io/_uploads/H1WuRCEy6.png)

Tiến hành đào sâu vào bên trong hàm này:

![](https://hackmd.io/_uploads/H1Xc00Vyp.png)

Có thể thấy được sơ lược qua, đầu tiên nó sẽ cop một file có nội dung là `CrazyArcade.sys` tiếp đến nó load số lượng ký tự vào buffer rồi tiếp đến nó đang bảo phải chạy dưới quyền admin. Tiếp đến một `hDevice` được tạo, ta đã biết được thường các `hDevice` tạo ra nhằm mục đích để chuyển hướng chương trình đang chạy đến một chương trình khác.

![](https://hackmd.io/_uploads/Sk8rlJSka.png)

Tiếp đến, chương trình load một lượng dữ liệu vào buffer, ta có thể nhận biết được bằng symbol [DeviceIoControl](https://learn.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-deviceiocontrol), sau cùng thì chương trình load những hiệu ứng của trò chơi vào và chạy chương trình. 

Debug chương trình này, bắt gặp được đoạn code để chạy con game nằm ở đây:

![](https://hackmd.io/_uploads/rymzZkry6.png)

![](https://hackmd.io/_uploads/rydIb1HJp.png)

Bên trong hàm này, nếu debug hoàn toàn thì chỉ có thể thấy được dường như nó chỉ chạy chương trình, và chả có gì đặc biệt ở đây. 

Nhưng ta sẽ thử tham khảo thử `DeviceIoControl` nó được lấy từ đâu, bởi vì nếu chơi chiến thắng game có flag thì có lẽ weight đã không tới 100 =))) và hơn thế nữa tại sao lại chỉ trò chơi, nhưng điều khiển chương trình thực thi một chương trình khác làm gì ??

Đây là kết quả nhận lại được:

![](https://hackmd.io/_uploads/S1QlMyrJa.png)

Có thể thấy chỉ có 2 hàm duy nhất của chương trình chứa `DeviceIoControl` đó là hàm đang phân tích và một hàm nữa có tên `sub_7FF6930B30A0` *(vì chương trình đã debug nên nó đã cộng thêm imagebase nhưng có thể check bằng 4 số cuối)*

Khi vào hàm này và đây là kết quả:

![](https://hackmd.io/_uploads/S15vMkHkT.png)

Ở đây, hình như đang thực hiện xor một thứ gì đó, static đối với những người lâu năm thì có lẽ khá dễ dàng nhận ra, nhưng với mình mình sẽ debug đoạn này để xem nó đang xor cái gì ở đây.

![](https://hackmd.io/_uploads/HJYDHyr1T.png)

Đầu tiên nó đang load một dữ liệu từ đâu đó

![](https://hackmd.io/_uploads/rJYtrkH1p.png)

Tiếp đến nó load dữ liệu được lấy từ đoạn phân tích với `DeviceIoControl` ở trên, sau cùng nó tiến hành 2 lần xor. Một lần xor với `r8d` và một lần xor với `ecx`.

> Ở đây mình đã cố tình patch lại để cho có thể nhảy trực tiếp vào đoạn xor này, có thể nói là không theo dự định của tác giả, vì sau nhiều lần debug kể cả có kill được những con bot hay thua hay di chuyển trong game thì mình vẫn ko thể debug theo ý muốn được, từng bước di chuyển hay khi giết hoặc chết mặc dù đang debug nhưng IDA vẫn không detect được. Ý định ban đầu muốn thử xem các sự kiện trong game sau mỗi lần thao tác có dừng lại để chạy asm được hay không ?? Nhưng không thể.

Vậy công việc cuối cùng bây giờ ta phải xác định được dữ liệu khi load vào `ecx` được lấy từ đâu ? Nếu nhìn kĩ lại chương trình này nó còn gọi một chương trình khác có cùng tên nhưng file là `.sys`. Ta sẽ mở file này lên xem có gì ở đây.

![](https://hackmd.io/_uploads/SyWOuJByT.png)

File này có rất nhiều hàm nhưng đa số được sử dụng hàm `sub_11450`. Nếu đã từng làm ở giải SEKAI, thì ở giải đó chính là phân tích thẳng kernel, đây cũng tương tự, nó đang tạo ra một drive để thực hiện một số điều gì đó, ta kiểm tra hàm được sử dụng nhiều nhất và bump:

![](https://hackmd.io/_uploads/S1d8Y1Sy6.png)

Thật là quen :))) vậy tất cả đã rõ, bây giờ chỉ còn việc viết script để hoàn tất:

```python
with open('CrazyArcade.sys', 'rb') as file:
    bytes = file.read()
    index = bytes.find(b'\x48\x89\x54\x24\x10') 
    print(hex(index))

flag = bytearray(b'\xb7\x8a\x19\x7fT-\x81\xf0\xb8\xdd\xca\xc9\xd3\xc3#2\xbaA\x81\xab\x02S\xc9.\xd6~ \xad\xab\xed\x95\xd2\xb6\xe7*\x92>')
sth = open("./CrazyArcade.sys", "rb").read()

sth = sth[0x850:]
r8d = 0

for _ in range(len(sth)):
    for i in range(37):
        flag[i] ^= (r8d ^ sth[r8d % 0x584])&0xff
        r8d += 1
    
    if b'hitcon{' in flag:
        print(flag)
        exit(0)
```

## The-blade (Rust)

*Đây là một chương trình được viết bằng rust - một ngôn ngữ khá trầm cảm khi rev. Nếu cho mình xếp độ khó thì mình chọn bài này khó nhất và hay nhất, bởi mình đã mất rất nhiều ngày để hoàn thành được nó, mặc dù sau giải :(((*

Không như những bài khác, đối với bài này, mình không làm tà đạo bằng cách mở thẳng IDA và chạy ngay, thay vào đó sẽ mở xem thử chương trình chạy thế nào. ~~Bởi có mở IDA liền thì cũng chả xác định được gì ngoài một núi hàm :))~~

Đây là những gì ta nhận được khi bắt đầu chạy chương trình:

![](https://hackmd.io/_uploads/BJLQmxzlp.png)

Tiếp đến, ta sẽ thử hết những lệnh ở đây, thấy được chỉ có mỗi `server` là thứ có vẻ ta nên tập trung vào.

![](https://hackmd.io/_uploads/SJY_meMxT.png)

Khi chạy server ta sẽ nhận được kết nối đến client(ở localhost).

![](https://hackmd.io/_uploads/r1eUEgGgp.png)

Ở đây, thấy được rằng có một đoạn shellcode được hiện ra ở server, ngoài ra khi có được shell thì ta chẳng thấy được điều gì khác nữa. Vậy câu hỏi đặt ra đó là flag ở đâu ?? làm gì để check flag khi có shell ??

Thử tất cả các cách từ debug đến dò từng hàm, vẫn không thấy bất cứ điều gì, sau cùng mình chỉ còn cách nghĩ đến `strings` nguyên file này thử xem có flag hay không. Điều này có vẻ khá ngớ ngẫn đối với các giải lớn nhưng vẫn cứ thử thôi. Và đây là thứ mình đã có được:

![](https://hackmd.io/_uploads/H13VwxMg6.png)

Mở IDA kiểm tra xem có gì ở đó không. Cứ theo hàm main ta sẽ thấy được một hàm như sau:

![](https://hackmd.io/_uploads/HJWA9gzl6.png)

> Tại sao ở đây có rất nhiều hàm nhưng mình chỉ focus vào cái này? Đó là bởi nếu đọc kỹ comment ta sẽ phần nào đoán được đoạn code này đang promt gì đó với shell, còn các hàm kia chỉ promt, print, display, color, ... trên server nên Không có gì đáng để quan tâm các hàm này.

Tiếp tục, cứ xem thử từng hàm thì ta sẽ thấy được một hàm rất khả nghi có tên `verify`. Nó khả nghi là bởi tất cả các hàm khác chỉ convert, up file, netcat,...(có thể để ý thêm tất cả các này đều hiển thị khi netcat thành công). Và hàm này có thể đoán được nó đang check flag, bởi một phần vì tên của nó, một phần vì nó không liên quan gì tới những thứ hiển thị khi ta có được shell.

![](https://hackmd.io/_uploads/Byee1bzep.png)

Đây chính là hàm checkflag:

![](https://hackmd.io/_uploads/Hy8lfZMga.png)

Kết hợp với dò từng hàm cũng như nhận thấy hàm checkflag, ta có thể rút ra được vài kết quả như sau: Chương trình chạy server bằng `server -> run -> nc`, sau đó sẽ check flag bằng lệnh `flag hitcon{...}`(với chiều dài flag là 64).

Sau khi xác định được tất cả những thứ đó, bây giờ ta sẽ tập trung vào hàm check này.

![](https://hackmd.io/_uploads/HJGvtpGx6.png)

Nhìn vào đoạn code, có rất nhiều `do while` ở đây, theo mình đã tiếp xúc, đối với những đoạn như này nếu thuật toán ngắn (đoạn code ngắn) như bài ở DucCTF(transform) thì mình có thể dịch lại và code lại để có thể dễ debug. Còn nếu đoạn code dài như ở bài này thì ta có thể đoán như bài ở giải SEKAI(Sahuang flag checker), đó là mỗi `do while` nó chia flag ra thành từng phần để tính toán và thao tác.

Như đã nói ở trên, ta sẽ đi từng phần. Đầu tiên, thấy được rất nhiều đoạn code khá giống nhau. Chúng thực hiện memcpy và sau đó swap một thứ gì đó. Để xem hoàn toàn nó thực sự đang làm gì ở đây mình sẽ tiến hành debug, với 2 breakpoint được đặt tại đầu và cuối cùng của đoạn code giống nhau này.

Đầu tiên ta thấy được đoạn code sau:

![](https://hackmd.io/_uploads/HyOFfCzep.png)

Ở đây đoạn ASM này đang tiến hành load input vào và sau đó tiến hành thao tác trên input.

Sau khi chạy xong đoạn memcpy và đến `do while` tiếp theo thì đây là kết quả đã thu được:

![](https://hackmd.io/_uploads/ryDV70zep.png)

Ta cùng nhau nhìn lại một xíu về đoạn này. Với đầu vào được đưa vào đó là `flag abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/` và đầu ra `Rp5v+AZmM8XWy1sgNhTB/oCzYVdPrGn6KD3Q9lke4qtFxHb0uUOcS2jIEJfL7aiw`. Đầu vào và đầu ra đều có chiều dài như nhau, ta lại thử tiếp một lần với input toàn là `a` và đây là kết quả:

![](https://hackmd.io/_uploads/BkMPEAMl6.png)

Tiếp tục với `abcd`, ta cũng có được output như sau:

![](https://hackmd.io/_uploads/S1gJr0zl6.png)

> Sở dĩ có thêm dấu `+/` là bởi có một hàm `memcpy(dest, "/", 0x200uLL);` mình cũng không thực sự nó có tác dụng gì nên mình thêm vào luôn input.

Vậy từ 3 kết quả trên ta thấy được `input[i]` đầu vào sẽ được map tương ứng với `input[j]` . Nên bây giờ ta sẽ lấy kết quả đầu tiên vì kết quả đầu tiên bao gồm `a...zA...Z0...9+\` hầu như đã có gần hết các ký tự có thể in được. Ta sẽ làm lại đoạn code map `do while` như sau:

```python
ip = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/"
resmap = "Rp5v+AZmM8XWy1sgNhTB/oCzYVdPrGn6KD3Q9lke4qtFxHb0uUOcS2jIEJfL7aiw"

mp = [0]*64

for idx, ch in enumerate(ip):
    mp[idx] = resmap.index(ch)
```
> Đoạn code trên đã làm lại đoạn map trong chương trình, có vẻ như nó giống với hoán vị hơn bởi mỗi vị trí ouput tương ứng chính là vị trí của đầu vào đã có.

Để rev lại đoạn code này khá dễ chỉ cần làm ngược lại là xong.

Tiếp đến đoạn tiếp theo:

![](https://hackmd.io/_uploads/B1FV5MQeT.png)

Đoạn code này, trong quá trình debug có thể thấy được từ kết quả hoán vị của phần trên thì với mỗi ký tự sau khi hoán vị nó được map thành một output nhất định tương ứng. Lúc đầu nhìn thấy chỉ xor và cộng trừ nên mình đã thử viết lại nhưng đoạn `do` ở trên mình chưa nhìn được nó là gì nên mình chả viết lại để rev được. 

Khi không hiểu rõ thuật toán chương trình mà cũng không thể code lại được thuật toán đó thì mình chỉ còn cách chạy thẳng chương trình lưu kết quả trước và sau khi thực hiện.

Đối với lần đầu tiên thì chương trình lấy kết quả đã hoán vị của input để map:

![](https://hackmd.io/_uploads/HyIg6MXlT.png)

Lần tiếp theo chạy nó lại tiếp tục load những bytes đã set sẵn để map:

![](https://hackmd.io/_uploads/HJXETMXxT.png)

![](https://hackmd.io/_uploads/HkGrTzmeT.png)

Sau nhiều lần chạy với những input khác nhau, ta để ý mỗi ký tự sẽ được map với 1 số nhất định dù cho input có khác nhau đi nữa. Vậy bây giờ công việc của mình phải xem hết 256 bytes thì với mỗi bytes thì thuật toán map tương ứng với giá trị nào. Và đây là kết quả:

```python
valmap = [0x52, 0x70, 0x35, 0x76, 0x2B, 0x41, 0x5A, 0x6D, 0x4D, 0x38, 0x58, 0x57, 0x79, 0x31, 0x73, 0x67, 0x4E, 0x68, 0x54, 0x42, 0x2F, 0x6F, 0x43, 0x7A, 0x59, 0x56, 0x64, 0x50, 0x72, 0x47, 0x6E, 0x36, 0x4B, 0x44, 0x33, 0x51, 0x39, 0x6C, 0x6B, 0x65, 0x34, 0x71, 0x74, 0x46, 0x78, 0x48, 0x62, 0x30, 0x75, 0x55, 0x4F, 0x63, 0x53, 0x32, 0x6A, 0x49, 0x45, 0x4A, 0x66, 0x4C, 0x37, 0x61, 0x69, 0x77, 0x04, 0x05, 0x06, 0x07, 0x00, 0x01, 0x02, 0x03, 0x0C, 0x0D, 0x0E, 0x0F, 0x08, 0x09, 0x0A, 0x0B, 0x14, 0x15, 0x16, 0x17, 0x10, 0x11, 0x12, 0x13, 0x1C, 0x1D, 0x1E, 0x1F, 0x18, 0x19, 0x1A, 0x1B, 0x24, 0x25, 0x26, 0x27, 0x20, 0x21, 0x22, 0x23, 0x2D, 0x2E, 0x3A, 0x3B, 0x28, 0x29, 0x2A, 0x2C, 0x40, 0x5B, 0x5C, 0x5D, 0x3C, 0x3D, 0x3E, 0x3F, 0x7C, 0x7D, 0x7E, 0x7F, 0x5E, 0x5F, 0x60, 0x7B, 0x84, 0x85, 0x86, 0x87, 0x80, 0x81, 0x82, 0x83, 0x8C, 0x8D, 
0x8E, 0x8F, 0x88, 0x89, 0x8A, 0x8B, 0x94, 0x95, 0x96,0x97, 0x90, 0x91, 0x92, 0x93, 0x9C, 0x9D, 0x9E, 0x9F, 0x98, 0x99, 
0x9A, 0x9B, 0xA4, 0xA5, 0xA6, 0xA7, 0xA0, 0xA1, 0xA2, 0xA3, 0xAC, 0xAD, 0xAE, 0xAF, 0xA8, 0xA9, 0xAA, 0xAB, 0xB4, 0xB5, 
0xB6, 0xB7, 0xB0, 0xB1, 0xB2, 0xB3, 0xBC, 0xBD, 0xBE, 0xBF, 0xB8, 0xB9, 0xBA, 0xBB, 0xC4, 0xC5, 0xC6, 0xC7, 0xC0, 0xC1, 0xC2, 0xC3, 0xCC, 0xCD, 0xCE, 0xCF, 0xC8, 0xC9, 0xCA, 0xCB, 0xD4, 0xD5, 0xD6, 0xD7, 0xD0, 0xD1, 0xD2, 0xD3, 0xDC, 0xDD, 0xDE, 0xDF, 0xD8, 0xD9, 0xDA, 0xDB, 0xE4, 0xE5, 0xE6, 0xE7, 0xE0, 0xE1, 0xE2, 0xE3, 0xEC, 0xED, 0xEE, 0xEF, 0xE8, 0xE9, 0xEA, 0xEB, 0xF4, 0xF5, 0xF6, 0xF7, 0xF0, 0xF1, 0xF2, 0xF3, 0xFC, 0xFD, 0xFE, 0xFF, 0xF8, 0xF9, 0xFA, 0xFB]
resmap = [0x58, 0x6C, 0x61, 0x2E, 0x69, 0x32, 0xCB, 0xE2, 0xB3, 0xE0, 0x02, 0xA0, 0x86, 0x1C, 0x6B, 0xC1, 0xEC, 0x9C, 0x79, 0xD2, 0x9E, 0xC2, 0xD9, 0x74, 0x0C, 0x3B, 0x04, 0x9F, 0x1E, 0x03, 0x14, 0xED, 0xA2, 0x8F, 0x97, 0xCA, 0xDA, 0xD8, 0xA4, 0x39, 0x5B, 0x64, 0x7E, 0xAF, 0x0B, 0x93, 0x71, 0x0F, 0x99, 0xFD, 0x81, 0x0A, 0xB7, 0x66, 0xEF, 0x3A, 0xEE, 0x00, 0xFF, 0xE1, 0xAD, 0x75, 0xAB, 0x09, 0x51, 0x15, 0x8D, 0xDB, 0xFB, 0x7B, 0x4E, 0xBB, 0xAA, 0xB2, 0x60, 0xEB, 0xB0, 0xAC, 0xA5, 0x8E, 0x2B, 0xC6, 0xA6, 0x35, 0x63, 0x5C, 0xDE, 0x42, 0xBD, 0x24, 0xB1, 0xE3, 0x30, 0x43, 0xD6, 0x5F, 0x7C, 0x6D, 0x8B, 0x17, 0x8C, 0xA7, 0xD5, 0x2A, 0x59, 0xA9, 0x27, 0x06, 0x9D, 0x83, 0xFE, 0x10, 0x41, 0xA8, 0x80, 0xC0, 0x25, 0xDC, 0x5E, 0xE7, 0xC4, 0x2D, 0x4F, 0xF9, 0x16, 0x4D, 0x2F, 0x6A, 0x89, 0x6F, 0x5D, 0xE8, 0xFA, 0x94, 0xB6, 0x1F, 0x88, 0xC5, 
0x7F, 0x77, 0xEA, 0xB5, 0x5A, 0x65, 0x3F, 0xF4, 0x48,0x47, 0x11, 0xCF, 0xF1, 0x1B, 0xE9, 0x62, 0x6E, 0xB4, 0x12, 0xE4, 
0xBA, 0xDF, 0x4B, 0x28, 0xD7, 0xD1, 0x96, 0xCD, 0x13, 0x53, 0x2C, 0x9B, 0x29, 0x44, 0x33, 0xB8, 0xE6, 0x7A, 0x31, 0xD3, 
0xB9, 0x40, 0x52, 0xF7, 0x20, 0xF2, 0x1A, 0x01, 0xA1, 0x92, 0xD0, 0x34, 0xF5, 0x54, 0xDD, 0xBC, 0x19, 0xF3, 0xFC, 0x85, 0x07, 0xBE, 0x4C, 0x7D, 0xC7, 0xD4, 0x36, 0xF6, 0x72, 0x98, 0x8A, 0xE5, 0x50, 0x46, 0x45, 0x4A, 0x9A, 0xC3, 0xC9, 0x0E, 0x3C, 0x57, 0xCC, 0x68, 0x76, 0x67, 0x84, 0x0D, 0x90, 0xA3, 0xF0, 0x22, 0xBF, 0x26, 0x91, 0x05, 0x87, 0x70, 0xAE, 0x3D, 0x1D, 0xC8, 0x55, 0x3E, 0x37, 0x23, 0x08, 0x73, 0x21, 0x49, 0x38, 0x95, 0x78, 0xF8, 0x18, 0x56, 0xCE, 0x82]

mp = [0]*256
rmp = [0]*256

for a,b in zip(valmap, resmap):
    mp[a] = b
```
> Ở đây, ta sẽ lợi dụng IDA để làm việc này, ta thấy được nội dung được lưu ở heap nên ta sẽ vào thẳng heap để chỉnh lại từng bytes sau đó lưu lại và chạy tiếp chương trình thì sẽ thu được kết quả tương ứng

*Sau khi xong được 2 phần trên, sau cùng là đoạn mình cho là hay nhất trong bài.*

Cuối cùng là đoạn shellcode:

![](https://hackmd.io/_uploads/BkewTEXea.png)

Sở dĩ tại sao nhận ra được đây là đoạn shellcode là bởi debug thì ở client nhận được một đoạn bytes, ta chỉ việc lấy những bytes đó đưa về shellcode. Và đây là đoạn code thực hiện điều này:

```cpp
#include <stdio.h>
#include <stdlib.h>

unsigned char shellcode[] = {0x54, 0x5d, 0x31, 0xf6, 0x48, 0xb9, 0xa1, 0x57, 0x06, 0xb8, 0x62, 0x3a, 0x9f, 0x37, 0x48, 0xba, 0x8e, 0x35, 0x6f, 0xd6, 0x4d, 0x49, 0xf7, 0x37, 0x48, 0x31, 0xd1, 0x51, 0x54, 0x5f, 0x6a, 0x02, 0x58, 0x99, 0x0f, 0x05, 0x48, 0x97, 0x31, 0xc0, 0x50, 0x54, 0x5e, 0x6a, 0x04, 0x5a, 0x0f, 0x05, 0x41, 0x5c, 0x6a, 0x03, 0x58, 0x0f, 0x05, 0x31, 0xf6, 0x48, 0xb9, 0x3b, 0x3b, 0x6f, 0xc3, 0x63, 0x64, 0xc0, 0xaa, 0x48, 0xba, 0x48, 0x4c, 0x0b, 0xc3, 0x63, 0x64, 0xc0, 0xaa, 0x48, 0x31, 0xd1, 0x51, 0x48, 0xb9, 0x8c, 0x57, 0x82, 0x75, 0xd6, 0xf8, 0xa9, 0x7d, 0x48, 0xba, 0xa3, 0x32, 0xf6, 0x16, 0xf9, 0x88, 0xc8, 0x0e, 0x48, 0x31, 0xd1, 0x51, 0x54, 0x5f, 0x6a, 0x02, 0x58, 0x99, 0x0f, 0x05, 0x48, 0x97, 0x31, 0xc0, 0x50, 0x54, 0x5e, 0x6a, 0x04, 0x5a, 0x0f, 0x05, 0x41, 0x5d, 0x6a, 0x03, 0x58, 0x0f, 0x05, 0x31, 0xf6, 0x6a, 0x6f, 0x48, 0xb9, 0x59, 0xe5, 0x06, 0x0c, 0x2d, 0xf6, 0xd9, 0x77, 0x48, 0xba, 0x76, 0x81, 0x63, 0x7a, 0x02, 0x8c, 0xbc, 0x05, 0x48, 0x31, 0xd1, 0x51, 0x54, 0x5f, 0x6a, 0x02, 0x58, 0x99, 0x0f, 0x05, 0x48, 0x97, 0x31, 0xc0, 0x50, 0x54, 0x5e, 0x6a, 0x04, 0x5a, 0x0f, 0x05, 0x58, 0x48, 0xf7, 0xd0, 0x48, 0xc1, 0xe8, 0x1d, 0x48, 0x99, 0x6a, 0x29, 0x59, 0x48, 0xf7, 0xf1, 0x49, 0x96, 0x6a, 0x03, 0x58, 0x0f, 0x05, 0xb8, 0xef, 0xbe, 0xad, 0xde, 0x44, 0x01, 0xe0, 0x44, 0x31, 0xe8, 0xc1, 0xc8, 0x0b, 0xf7, 0xd0, 0x44, 0x31, 0xf0, 0x3d, 0xef, 0xbe, 0xad, 0xde, 0x75, 0x05, 0x6a, 0x01, 0x58, 0xeb, 0x03, 0x48, 0x31, 0xc0, 0x50, 0x53, 0x5f, 0x54, 0x5e, 0x6a, 0x08, 0x5a, 0x6a, 0x01, 0x58, 0x0f, 0x05, 0x55, 0x5c, 0x41, 0xff, 0xe7};
int main()
{
    ((void (*)(void))(shellcode))();
}
```
> gcc main.c -z execstack -o main

Sau khi có được binary của đoạn shellcode ta lại ném vào IDA:

![](https://hackmd.io/_uploads/H1CHCNXeT.png)

Khi nhìn vào đoạn code này có vẻ như có một số điều khá thú vị ở đây. Đoạn code này đọc 4 bytes từ `/bin/sh` và `etc/passwd` sau đó tiếp tục đọc tại `/dev/zero` cuối cùng thực hiện các phép toán (các phép toán này có thể rev được). Vậy bây giờ ta cần xem xem 4 bytes ở đây là gồm những gì:

```cpp
#include <stdio.h>

int main() {
    FILE *file;
    unsigned char buffer[4];
    
    file = fopen("/dev/zero", "rb");
    if (file == NULL) {
        printf("cant open.\n");
        return 1;
    }
    
    fread(buffer, sizeof(unsigned char), 4, file);
    
    printf("4 bytes /etc/passwd: %02x %02x %02x %02x\n", buffer[0], buffer[1], buffer[2], buffer[3]);
    
    fclose(file);
    
    return 0;
}
```
> Thay lần lượt những đường dẫn mình cần đọc

Vậy từ đó ta đã có hết tất cả các giá trị. Cuối cùng rev chúng và kết hợp chúng lại. Đây là đoạn code để giải bài này:


```python
valMp = [0x52, 0x70, 0x35, 0x76, 0x2B, 0x41, 0x5A, 0x6D, 0x4D, 0x38, 0x58, 0x57, 0x79, 0x31, 0x73, 0x67, 0x4E, 0x68, 0x54, 0x42, 0x2F, 0x6F, 0x43, 0x7A, 0x59, 0x56, 0x64, 0x50, 0x72, 0x47, 0x6E, 0x36, 0x4B, 0x44, 0x33, 0x51, 0x39, 0x6C, 0x6B, 0x65, 0x34, 0x71, 0x74, 0x46, 0x78, 0x48, 0x62, 0x30, 0x75, 0x55, 0x4F, 0x63, 0x53, 0x32, 0x6A, 0x49, 0x45, 0x4A, 0x66, 0x4C, 0x37, 0x61, 0x69, 0x77, 0x04, 0x05, 0x06, 0x07, 0x00, 0x01, 0x02, 0x03, 0x0C, 0x0D, 0x0E, 0x0F, 0x08, 0x09, 0x0A, 0x0B, 0x14, 0x15, 0x16, 0x17, 0x10, 0x11, 0x12, 0x13, 0x1C, 0x1D, 0x1E, 0x1F, 0x18, 0x19, 0x1A, 0x1B, 0x24, 0x25, 0x26, 0x27, 0x20, 0x21, 0x22, 0x23, 0x2D, 0x2E, 0x3A, 0x3B, 0x28, 0x29, 0x2A, 0x2C, 0x40, 0x5B, 0x5C, 0x5D, 0x3C, 0x3D, 0x3E, 0x3F, 0x7C, 0x7D, 0x7E, 0x7F, 0x5E, 0x5F, 0x60, 0x7B, 0x84, 0x85, 0x86, 0x87, 0x80, 0x81, 0x82, 0x83, 0x8C, 0x8D, 
0x8E, 0x8F, 0x88, 0x89, 0x8A, 0x8B, 0x94, 0x95, 0x96, 0x97, 0x90, 0x91, 0x92, 0x93, 0x9C, 0x9D, 0x9E, 0x9F, 0x98, 0x99, 0x9A, 0x9B, 0xA4, 0xA5, 0xA6, 0xA7, 0xA0, 0xA1, 0xA2, 0xA3, 0xAC, 0xAD, 0xAE, 0xAF, 0xA8, 0xA9, 0xAA, 0xAB, 0xB4, 0xB5, 
0xB6, 0xB7, 0xB0, 0xB1, 0xB2, 0xB3, 0xBC, 0xBD, 0xBE, 0xBF, 0xB8, 0xB9, 0xBA, 0xBB, 0xC4, 0xC5, 0xC6, 0xC7, 0xC0, 0xC1, 0xC2, 0xC3, 0xCC, 0xCD, 0xCE, 0xCF, 0xC8, 0xC9, 0xCA, 0xCB, 0xD4, 0xD5, 0xD6, 0xD7, 0xD0, 0xD1, 0xD2, 0xD3, 0xDC, 0xDD, 0xDE, 0xDF, 0xD8, 0xD9, 0xDA, 0xDB, 0xE4, 0xE5, 0xE6, 0xE7, 0xE0, 0xE1, 0xE2, 0xE3, 0xEC, 0xED, 0xEE, 0xEF, 0xE8, 0xE9, 0xEA, 0xEB, 0xF4, 0xF5, 0xF6, 0xF7, 0xF0, 0xF1, 0xF2, 0xF3, 0xFC, 0xFD, 0xFE, 0xFF, 0xF8, 0xF9, 0xFA, 0xFB]
resMp = [0x58, 0x6C, 0x61, 0x2E, 0x69, 0x32, 0xCB, 0xE2, 0xB3, 0xE0, 0x02, 0xA0, 0x86, 0x1C, 0x6B, 0xC1, 0xEC, 0x9C, 0x79, 0xD2, 0x9E, 0xC2, 0xD9, 0x74, 0x0C, 0x3B, 0x04, 0x9F, 0x1E, 0x03, 0x14, 0xED, 0xA2, 0x8F, 0x97, 0xCA, 0xDA, 0xD8, 0xA4, 0x39, 0x5B, 0x64, 0x7E, 0xAF, 0x0B, 0x93, 0x71, 0x0F, 0x99, 0xFD, 0x81, 0x0A, 0xB7, 0x66, 0xEF, 0x3A, 0xEE, 0x00, 0xFF, 0xE1, 0xAD, 0x75, 0xAB, 0x09, 0x51, 0x15, 0x8D, 0xDB, 0xFB, 0x7B, 0x4E, 0xBB, 0xAA, 0xB2, 0x60, 0xEB, 0xB0, 0xAC, 0xA5, 0x8E, 0x2B, 0xC6, 0xA6, 0x35, 0x63, 0x5C, 0xDE, 0x42, 0xBD, 0x24, 0xB1, 0xE3, 0x30, 0x43, 0xD6, 0x5F, 0x7C, 0x6D, 0x8B, 0x17, 0x8C, 0xA7, 0xD5, 0x2A, 0x59, 0xA9, 0x27, 0x06, 0x9D, 0x83, 0xFE, 0x10, 0x41, 0xA8, 0x80, 0xC0, 0x25, 0xDC, 0x5E, 0xE7, 0xC4, 0x2D, 0x4F, 0xF9, 0x16, 0x4D, 0x2F, 0x6A, 0x89, 0x6F, 0x5D, 0xE8, 0xFA, 0x94, 0xB6, 0x1F, 0x88, 0xC5, 
0x7F, 0x77, 0xEA, 0xB5, 0x5A, 0x65, 0x3F, 0xF4, 0x48, 0x47, 0x11, 0xCF, 0xF1, 0x1B, 0xE9, 0x62, 0x6E, 0xB4, 0x12, 0xE4, 
0xBA, 0xDF, 0x4B, 0x28, 0xD7, 0xD1, 0x96, 0xCD, 0x13, 0x53, 0x2C, 0x9B, 0x29, 0x44, 0x33, 0xB8, 0xE6, 0x7A, 0x31, 0xD3, 
0xB9, 0x40, 0x52, 0xF7, 0x20, 0xF2, 0x1A, 0x01, 0xA1, 0x92, 0xD0, 0x34, 0xF5, 0x54, 0xDD, 0xBC, 0x19, 0xF3, 0xFC, 0x85, 0x07, 0xBE, 0x4C, 0x7D, 0xC7, 0xD4, 0x36, 0xF6, 0x72, 0x98, 0x8A, 0xE5, 0x50, 0x46, 0x45, 0x4A, 0x9A, 0xC3, 0xC9, 0x0E, 0x3C, 0x57, 0xCC, 0x68, 0x76, 0x67, 0x84, 0x0D, 0x90, 0xA3, 0xF0, 0x22, 0xBF, 0x26, 0x91, 0x05, 0x87, 0x70, 0xAE, 0x3D, 0x1D, 0xC8, 0x55, 0x3E, 0x37, 0x23, 0x08, 0x73, 0x21, 0x49, 0x38, 0x95, 0x78, 0xF8, 0x18, 0x56, 0xCE, 0x82]

ip = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/'

resmp = 'Rp5v+AZmM8XWy1sgNhTB/oCzYVdPrGn6KD3Q9lke4qtFxHb0uUOcS2jIEJfL7aiw'

mp = [0]*64 

for idx, ch in enumerate(ip):
    mp[idx] = resmp.index(ch)

revMp = [0]*256 
for a, b in zip(valMp, resMp):
    revMp[b] = a

def revMap(a):
    return bytes([revMp[i] for i in a])

def revmap(b):
    tmp = [0]*64
    for i, j in enumerate(mp):
        tmp[i] = b[j]
    return bytes(tmp)

MASK32 = 0xFFFF_FFFF
def rol(x, n):
    return ((x << n) & MASK32) | (x >> (32 - n))

sh = 0x464c457f
passwd = 0x746f6f72
zero = (0xffffffffffffffff >> 29) // 41
enc = [0x526851a7,0x31ff2785,0xc7d28788,0x523f23d3,0xaf1f1055,0x5c94f027,0x797a3fcd,0xe7f02f9f,0x3c86f045,0x6deab0f9,0x91f74290,0x7c9a3aed,0xdc846b01,0x0743c86c,0xdff7085c,0xa4aee3eb]

flag = b''
for c in enc:
    c = c^zero
    c = (~c)&MASK32
    c = rol(c,11)
    c = ((c ^ passwd) - sh )&MASK32
    flag += c.to_bytes(4, 'little')

for i in range(256):
    flag = revMap(flag)
    flag = revmap(flag)

print(flag)
```