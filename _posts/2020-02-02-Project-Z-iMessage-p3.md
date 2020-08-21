---
title: "Project Zero: Remote iPhone Exploitation [P3]"
author: Huy
date: 2020-02-02 14:30:59 +0800
categories: [Series, "Project Zero"]
tags: [translate, reverse engineer, 0day]
---

Bài này được mình dịch từ blog của team [Google Project Zero](https://googleprojectzero.blogspot.com/2020/01/remote-iphone-exploitation-part-3.html).

Đây là phần cuối trong series exploit iMessage trên iOS. Ở phần một ta đã giới thiệu sơ lược về lỗ hổng và phần hai đi sâu vào các phương pháp tấn công như heapspray, leak base address, ...

Tới phần ba này, như ta đã biết, ASLR đã được bypass, base address của shared cache đã nằm trong tay và controlled data có thể được xếp vào một địa chỉ chính xác. Điều còn lại ta cần làm là exploit lỗ hổng thêm một lần nữa để chiếm quyền điều khiển.

**Sơ lược**: Mở đầu phần ba sẽ giới thiệu ngắn về việc sử dụng ObjC để khai thác các thiết bị không có xác thực con trỏ (Pointer Authentication - PAC), tiếp đó là đi sâu vào một cách khai thác có thể hoạt động kể cả khi PAC được kích hoạt. Cuối cùng sẽ là kỹ thuật xâu chuỗi lại các kiểu tấn công với Javascript kernel exploit.

# Objective-C Remote Code Executer

ObjC là một superset của C với các tính năng lập trình hướng đối tượng. ObjC cho chúng ta concept về đối tượng, các lớp với method và property cũng như tính kế thừa cho ngôn ngữ. Hầu hết các đối tượng trong ObjC đều kế thừa từ một lớp cơ sở là `NSObject`. Sau đây là một code snippet đơn giản của ObjC:

```objc
  Bob* bob = [[Bob alloc] init];
  [bob doSomething];
```

Đoạn snippet trên khởi tạo một instance của một lớp, sau đó gọi một method.

ObjC quản lý rất chặt đối với thời hạn (lifetime) của các object. Vì thế, mọi object đều có một bộ đếm tham chiếu (refcount) tồn tại [inline](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-private.h#L93) hoặc trong một [global table](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/NSObject.mm#L201). Khi làm việc với một object, đoạn code phải thực thi hai lệnh gọi là [obj_retain](https://developer.apple.com/documentation/objectivec/1418956-nsobject/1571946-retain?language=objc) và [objc_release](https://developer.apple.com/documentation/objectivec/1418956-nsobject/1571957-release?language=objc) lên object. Hai hàm này phải được thêm thủ công bởi lập trình viên hoặc [tự động bằng compiler](https://en.wikipedia.org/wiki/Automatic_Reference_Counting).

Trong nội bộ, một object của ObjC là một chunk memory (thường được khởi tạo bằng `alloc`) luôn bắt đầu với một [giá trị "ISA"](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-private.h#L59) và theo sau bởi các biến hoặc property. Giá trị ISA là một giá trị kiểu con trỏ có chứa thông tin sau:

- Một con trỏ đến Lớp của object
- Một inline refcount
- Một vài flag bit

Một [lớp ObjC](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-runtime-new.h#L1012) mô tả instance của nó và chính nó. Các lớp trong ObjC không chỉ là một khái niệm compile-time mà chúng cũng tồn tại ở runtime. Một lớp chứa các thông tin sau:

- Chữ ISA (do nó là một ObjC Object), trỏ đến một metaclass tương ứng
- Một con trỏ trỏ đến một lớp cha nếu có
- Một method cache để tăng tốc method implementation lookups
- Method table
- Danh sách các instance variable (tên và offset)

Một [method](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-runtime-new.h#L207) trong ObjC đơn giản là một tuple: 

([Selector](https://developer.apple.com/documentation/objectivec/sel?language=objc), [Implementation](https://developer.apple.com/documentation/objectivec/objective-c_runtime/imp?language=objc))

với Selector là một chuỗi C-String duy nhất chứa tên của method và Implementation là một con trỏ đến native function của method.

Dưới đây là những gì sẽ xảy ra khi đoạn code khi nãy được compile và thực thi: Đầu tiên, ở compile-time, ba lệnh gọi method ObjC được dịch sang một đoạn pseudo-C (giả sử ARC đang được kích hoạt):

```pseudocode
Bob* bob = objc_msgSend(BobClass, "alloc");
// Refcount is already 1 at this point, so no need for a objc_retain()
bob = objc_msgSend(bob, "init");
objc_msgSend(bob, "doSomething");
...
objc_release(bob);
```

Trong runtime, [objc_msgSend](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/Messengers.subproj/objc-msg-arm64.s#L252) sẽ thực thi: 

1. Truy cập vào vùng nhớ của con trỏ để thu giá trị ISA và extract Class pointer
2. Thực hiện việc [tìm kiếm method implementation](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-runtime-new.mm#L4657) trong method cache của Lớp.
3. Nếu bước 2 thất bại, tiếp tục tìm method trong method table
4. Nếu một method Implementation được tìm tháy, nó sẽ được gọi (đuôi - Tailed Call) với những tham số từ objc_msgSend. Nếu fail thì sẽ throw exception.

Lưu ý rằng vì các selector là unique trong runtime, so sánh hai selector với nhau có thể được đơn giản bằng việc so sánh giá trị của hai con trỏ tương ứng,

Mặt khác, [objc_release](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/NSObject.mm#L1505) sẽ [giảm refcount](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-object.h#L464) của object. Nếu như refcount của object đã bằng không thì nó sẽ [gọi](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-object.h#L571) method [dealloc_method](https://developer.apple.com/documentation/objectivec/nsobject/1571947-dealloc) (deconstructor của object) và sau đó giải phóng object memory chunk.

Một số thông tin khác có thể tìm thấy ở [đây](http://www.phrack.org/issues/66/4.html#article), [đây](http://www.phrack.org/issues/69/9.html#article) và trong [source](https://github.com/opensource-apple/objc4)/[binary](https://ipsw.me/) code.

## Native Code Execution trên các Thiết bị non-PAC

Qua những kiến thức có được về ObjC trên kết hợp với những thứ có được ở phần hai của series, ta có thể implement một exploit đơn giản để trigger NCE trên các thiết bị non-PAC, nổi bật là iPhone X và những đời trước.

Sau đây là một đoạn code nhỏ từ `[NSSharedKeyset indexForKey:]`. Đoạn code này sẽ lược qua các giá trị phù hợp do người tấn công chọn (với `_keys` là nullptr và `index `đã được kiểm soát) và xử lý chúng.

```objective-c
id candidate = self->_keys[index];
  if (candidate != nil) {
    if ([key isEqual:candidate]) {
      return prevLength + index;
    }
  }
```
Nếu key là môt NSString, thì điều đầu tiên `[NSString isEqual:]` làm là sẽ gọi `[arg isNSString__]` lên key. NCE có thể được trigger như sau:

1. Thực thi heapspray như ở phần hai. Heapsray sẽ chưa một fake method cache và fake object pointer
2. Trigger lỗ hổng để đọc con trỏ trên sau đó pass nó qua `[key isEqual:]`. Việc này sẽ invoke `isNSString__` lên fake object mà ta đã tạo từ bước đầu tiên, sau đó trỏ về address của attacker.
3. Pivot stack và implement payload vào trong ROP

Tại thời điểm này, người tấn công đã thành công trong việc exploit một thiết bị không hỗ trợ PAC. Tuy nhiên, khi PAC được kích hoạt, kiểu tấn công trên sẽ không còn phù hợp nữa.

## Ảnh hưởng của PAC trong Userspace

Nhà nghiên cứu bảo mật Brandon Azad đã giải thích rất tốt về [cách hoạt động của PAC trong kernel](https://googleprojectzero.blogspot.com/2019/02/examining-pointer-authentication-on.html). PAC trong userspace cũng không quá khác biệt.  Phần sau đây sẽ tóm tắt cách hoạt động của PAC trong userspace. Để biết thêm chi tiết, [bấm vào đây](https://github.com/apple/llvm-project/blob/apple/master/clang/docs/PointerAuthentication.rst).

Mỗi code pointer  (function pointer, method pointer, ...) đều có một secret key được signed cũng như là một optional 64-bit "context".  Signature trả về sẽ bị truncate xuống 24 bit sau đó được lưu trong phần trên của con trỏ, thường không được đụng đến. Trước khi truy cập vào code pointer, signature của nó sẽ được xác minh bằng cách recompute nó với cùng một key, context . Nếu chúng không trùng nhau, con trỏ sẽ trở nên rời rạc và dẫn đến access violation khi nó được truy cập. Ngược lại, nếu con trỏ trùng nhau, signature bit sẽ được xóa và con trỏ sẽ trở nên hợp lệ.

