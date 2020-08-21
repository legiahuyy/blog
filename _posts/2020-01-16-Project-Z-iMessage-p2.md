---
title: "Project Zero: Remote iPhone Exploitation [P2]"
author: Huy
date: 2020-01-16 9:11:55 +0800
categories: [Series, "Project Zero"]
tags: [translate, reverse engineer, 0day]
---

Bài viết này được mình dịch từ blog của team [Google Project Zero](https://googleprojectzero.blogspot.com/2020/01/remote-iphone-exploitation-part-2.html).

Đây là phần thứ hai của series khai thác lỗ hổng CVE-2019–8641 của app iMessage trên iOS 12.4. Nếu bạn chưa hiểu hoặc quên mất chuyện gì đang xảy ra, hãy xem lại [phần trước](https://medium.com/@ZeroUnix/project-zero-remote-iphone-exploitation-p1-85ed8aeddf51).

Lúc trước chúng ta đang nói đến đoạn tìm address space của thiết bị. Bài viết lần này sẽ xoay quanh việc bypass ASLR mà không cần phải có sự tương tác của người dùng hay phải tìm thêm lỗ hổng nào cả.

Đầu tiên ta sẽ bàn về một kỹ thuật khá “xưa cũ”: **heap spraying**. Kỹ thuật này sẽ giúp ta tìm ra được *base_address* của [DYLD_SharedCached_Region](http://www.manpagez.com/man/1/dyld/) (một dynamic linker của iOS) dựa trên lỗi tràn bộ nhớ.

# Heap spraying trên iOS

Mục đích của việc sử dụng heap spraying là để đưa data vào một vùng địa chỉ biết trước và tham chiếu đến nó khi khai thác. Heap spraying ngày nay cũng không còn hiệu quả nữa (15 năm tuổi rồi) vì sự có mặt của ASLR và đa phần các máy đều sử dụng 64-bit architecture. Tuy nhiên, trên iOS thì kỹ thuật này vẫn có thể xài được bình thường và cũng chỉ cần khoảng 256MB data để có thể chuyển control data vào vùng địa chỉ trên thiết bị. Hãy xem đoạn code sau:

```c++
const size_t size = 0x4000;
const size_t count = (256 * 1024 * 1024) / size;
for (int i = 0; i < count; i++) {
     int* chunk = malloc(size);
     *chunk = 0x41414141;
}
   // Giờ hãy để ý vùng địa chỉ 0x110000000 trong bộ nhớ
```

Sau khi thực thi thì đoạn code sẽ set giá trị tại `0x110000000` thành ` 0x41414141`:

```pseudocode
(lldb) x/gx 0x110000000
0x110000000: 0x0000000041414141
```

## Heap spraying iMessage

Điều tiếp theo chúng ta làm là tìm hiểu cách tốt nhất để “spray” vài trăm MB data qua iMessage. Nói đơn giản thì có hai cách để làm điều đó như sau:

1. Bằng cách lợi dụng bug memory leak — một bug mà chunk data bị “lãng quên” và không bao giờ được giải phóng, từ đó ta có thể khiến nó nhân lên vài lần cho đến khi tràn bộ nhớ.
2. Hoặc khai thác “amplification gadget”: một đoạn code nhỏ dùng để lấy chunk data có sẵn ra rồi copy => ta chỉ cần gửi một vài byte rồi để nó tự *khuếch đại* liên tục như thế để đạt được lượng byte mà ta muốn.

May mắn là NSKeyedUnarchiver API cũng “tài trợ” luôn cho chúng ta trong exploit lần này: Nó gửi được tin nhắn tầm 100kB là maximun rồi, vì vậy khi mình spray khoảng 32MB heap data nó sẽ leak ra 32MB và ta sẽ lặp lại điều đó một vài lần cho đến khi được full heap spray.

Nhưng mà ta có thể gửi toàn bộ số byte cần spray chỉ với một tin nhắn bằng cách sử dụng tính năng **download attachments** (pbdi key). Bằng cách này, memory leak là điều không cần thiết nữa và bug có thể được trigger vài lần nếu cần thiết. Mình sẽ dành phần này cho bạn đọc như một exercise nhé.

Có rất nhiều chỗ trong subsystem của NSKeyedUnarchive bị leak memory. Ví dụ như phần *cyclic object graph* trong phần một, khi mà chunk data không bao giờ được giải phóng mà cứ liên tục tham chiếu lẫn nhau. Hay một subclass khác là *__NSKeyedCoderOldStyleArray* (dùng để decode [NSValues](https://developer.apple.com/documentation/foundation/nsvalue)) lại luôn deserialize mọi class bất kể có được phép hay không. __NSKeyedCoderOldStyleArray sẽ lưu size và con trỏ của nó vào một chunk đã được khởi tạo dưới dạng cùng kiểu dữ liệu (integer, CString hoặc ObjC object).

Thú vị là khi __NSKeyedCoderOldStyleArray bị giải phóng, nó chỉ free đúng phần backing memory và không free object bên trong. Do đó, mảng chứa các con trỏ sẽ trỏ đến một vùng nhớ khác dẫn đến memory leak. Điều này đáng ra không quá tệ vì __NSKeyedCoderOldStyleArray chỉ được dùng để decode “tạm thời” một mảng của NSValue nhưng vì __NSKeyedCoderOldStyleArray cũng có thể *được decode* dưới dạng một standalone object khiến nó leak những ObjC object bên trong.

Ngoài ra phần *amplification gadget* cũng có một instance ACZeroingString cũng đóng vai trò quan trọng trong exploit vì nó là một phần của *initWithCoder*, ACZeroingString sẽ lấy một NSData object rồi copy nội dung vào một vùng nhớ mới.

Sau đây là graph của việc spray 32MB heap data:

![Image for post](https://miro.medium.com/max/800/0*n1CVMeFdgXWIO6lh.png)

Ta có thể thấy ở mỗi instance ACZeroingString nó sẽ copy nội dung của NSData vào một buffer và __NSKeyedCoderOldStyleArrray sẽ giữ toàn bộ instance đó được liên tục update kể cả khi tin nhắn đã được gửi đi. Sau khi gửi 8 tin nhắn như thế này, control data sẽ có địa chỉ tại *0x110000000*.

Việc chuyển control data vào địa chỉ trên chỉ là bước đầu tiên mà thôi. Tiếp theo chúng ta sẽ phải tạo các *đối tượng ObjC giả* trong vùng heapspray và để hệ thống xử lí chúng trong key index của NSSharedKeySet. Tuy nhiên, vì nội dung bên trong heapspray không thể chạy được, ta cần phải biết địa chỉ của code page trước khi làm giả object.

# Bypass ASLR bằng Objective-C

Dưới đây là một snippet của **[NSSharedKeySet indexForKey:]**

```pseudocode
// index is fully controlled, _keys is nullptr
      id candidate = self->_keys[index];
      if (candidate != null) {
        if ([key isEqual:candidate]) {
          return prevLength + index;
        }
      }
```

Kiểu dữ liệu ‘id’ được sử dụng ở đây là đại diện cho việc tham chiếu đến ObjC object, na ná `void*` trong C. Nhưng điều khác biệt là Object-C có RTTI (**R**un-**t**ime **T**ype **I**nformation) nên nó sẽ luôn biết được kiểu dữ liệu của một object trong runtime, ví dụ như [*isKindOfClass*](https://developer.apple.com/documentation/objectivec/1418956-nsobject/1418511-iskindofclass?language=objc) method. Hơn nữa, ObjC hỗ trợ pointer tagging (con trỏ địa chỉ), và object pointer (con trỏ đối tượng). Vì thế, ‘id’ có thể là một trong hai thứ sau:

1. Là một object pointer
2. Là một pointer-sized value chứa cả kiểu dữ liệu lẫn thông tin.

Phần layout của những object này rất quan trọng vì nó chính là ‘chìa khóa’ để ta có được RCE nhưng trong bài viết này ta sẽ khoan nhắc đến nó mà chỉ dừng ở phần tagged pointer trước đã.

[NSNumbers](https://developer.apple.com/documentation/foundation/nsnumber?language=objc) và [NSStrings](https://developer.apple.com/documentation/foundation/nsstring) là hai ví dụ cho tagged pointer. Trên bộ xử lí Arm64, một ‘id’ được xem như là một tagged pointer nếu [flag MSB](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-config.h#L79) của nó được set. Trong trường hợp đó, class tương ứng sẽ được [lưu trong một global table](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/objc-object.h#L656) và index của table đó sẽ được encode vào tagged pointer. Thế nên một instance NSNumber chứa một 32-bit integer (trong ObjC: `NSNumber* n = @42`) sẽ được biểu diễn như sau:

```
0xb0000000000002a2
(1 011 00…001010100010)
```

Khi đó MSB flag sẽ có giá trị (1), ra hiệu rằng một tagged pointer và 3 bit tiếp theo của nó là index của class. Nếu MSB flag là (3), tương ứng với __NSCFNumber, giá trị 32-bit của con trỏ sẽ nằm trong bit 8->40 trong khi các byte dưới cùng chỉ ra kiểu dữ liệu, trong trường hợp này là [kCFNumberSInt32Type](https://developer.apple.com/documentation/corefoundation/cfnumbertype/kcfnumbersint32type?language=objc).

Các API thường sẽ kiểm tra tag bit của các ‘id’ ObjC (objc_msgSend, objc_retain, objc_xyz):

```c
if (arg & 0x8000000000000000) {
 // handle tagged pointer
 } else {
 // handle real pointer
 }
```

Phương thức kiểm tra này là để chống các [kỹ thuật khai thác tagged pointer](http://www.phrack.org/issues/69/9.html), giá trị của các tagged pointer sẽ được XOR với một giá trị bất kì cho mỗi process:

```pseudocode
0xb0000000000002a2 ^ objc_debug_taggedpointer_obfuscator
```

`objc_debug_taggedpointer_obfuscator` có lẽ là một con số hoàn toàn ngẫu nhiên chỉ trừ MSB thì nó có giá trị là ‘0’ (bit dành riêng cho tagged pointer). Sử dụng lldb và một app iOS đơn giản ta có được như sau:

```objective-c
(lldb) p n
(__NSCFNumber *) $0 = 0xf460034a00975a82 (int)42
(lldb) p objc_debug_taggedpointer_obfuscator
(void *) $1 = 0x4460034a00975820
(lldb) p/x (uintptr_t)n ^ (uintptr_t)objc_debug_taggedpointer_obfuscator
(unsigned long) $9 = 0xb0000000000002a2
```

## Dyld Shared Cache

Trên iOS (hoặc macOS), hầu hết các thư viện hệ thống đều được prelink vào một binary blob rất lớn được gọi là [dyld_shared_cache](https://iphonedevwiki.net/index.php/Dyld_shared_cache). Điều này giúp cải thiện load-time rất đáng kể.

Mặt khác, shared cache được hệ thống map lại với cùng một địa chỉ cho mọi process và chỉ được random một lần duy nhất là khi thiết bị boot. Nguyên nhân có thể là do shared cache bị map vào toàn bộ process của user (làm giảm hiệu suất chung của bộ nhớ hệ thống) đồng thời nó cũng chứa một con trỏ của chính nó nữa khiến cho share cache có thể [được copy từ bất kì khu vực bộ nhớ nào](https://en.wikipedia.org/wiki/Position-independent_code). Nên ta sẽ biết được base_address dựa vào địa chỉ của shared cache, base_address sẽ chứa địa chỉ của hầu hết các thư viện mà các process của user đang sử dụng, bao gồm: các tiện ích ROP (gadget), mọi ObjC class, chuỗi và nhiều thứ khác. Quá đủ để thực hiện khai thác RCE.

Ở những phiên bản mới nhất của iOS, dyld_shared_cache sẽ được map lại đâu đó trong vùng từ 0x180000000 đén 0x280000000, bao gồm hơn 200000 base_address cũng như cache của nó cũng phải xấp xỉ 1GB và page size của nó khoẳng 0x4000 byte. Ta có thể tìm những base_address theo ý mình bằng bruteforce, nhưng mỗi lần sai có thể process sẽ bị crash và sẽ bị giới hạn bởi service *launchd* nếu bruteforce của ta làm máy crash quá nhiều lần. Điều này có thể được tránh bằng cách chỉ crash mỗi 10s => tốn đến 3–4 tuần để bruteforce — exploit kiểu này không khả thi lắm. Nhưng phần tiếp theo sẽ chỉ ra cách tìm base_address chỉ trong vòng chưa đến năm phút.

## Bypass ASLR với Oracle Crash

Một thứ rất đặc biệt của lỗ hổng CVE-2019–8646 là nó có thể tạo ra một communication channel giữa victim và attacker. Đây cũng là một tình huống nghiêm trọng mà các nhà phát hành nên lưu ý khi để các tiến trình tự do thực hiện các network request ngoài sandbox. Và để có được RCE nhắc đến ở phần một, ta phải tìm communication channel của iMessage.

iMessage cũng hỗ trợ các tính năng như “read” (đã xem), “delivered” (đã gửi) tương tự như seen hay sent của app Messenger của Facebook.

<img src="https://miro.medium.com/max/698/0*moeYbETbI9FYeskT.png" alt="Image for post" style="zoom:67%;" />

Người gửi nhận được tín hiệu ‘đã xem’ của người nhận ở tin nhắn “Foo” và tín hiệu ‘đã gửi’ ở tin nhắn “Bar” còn “Baz” thì không có gì cả. Các *tín hiệu* đó được gửi đi và nhận lại một cách tự động và không cần sự tương tác của ngời dùng, đặc biệt nó còn được gửi bởi tiến trình **imagent**.

Pseudocode cách imagent xử lí tin nhắn:

```pseudocode
processIncomingMessage(message):
msgPlist = decodeIntoPlist(message)
# extract some values from the plist …
atiData = msgPlist['ATI']
ati = NSKeyedUnarchive(atiData) [1]
# more stuff …
sendDeliveryReceipt()
# yet more stuff …
```

Bất kì bug nào trong NSKeyedUnarchiver API cũng sẽ được *kích hoạt* tại [1] nên ta có thể build một “oracle crash” nếu ta có thể crash khi API NSKeyedUnarchiver đang làm việc thì sẽ không có thông báo ‘delivery’ nào được gửi đi mà thay vào đó là một payload chúng ta muốn. Điều này cho phép *người gửi* có thể tùy ý crash tiến trình imagent của thiết bị nhận. Lợi dụng điều này, ta có thể chuyển từ một bug thành một “feature” có lợi cho việc tấn công rồi.

Ta có thể hiểu đơn giản các truy vấn oracle diễn ra như sau:

```python
def oracle(addr):
     if isMapped(addr):
       nocrash()
     else
       crash()
```

Nhờ đó, việc bypass ASLR từ xa trở nên đơn giản hơn:

1. Thực hiện [*tìm kiếm tuyến tính*](https://vietjack.com/cau-truc-du-lieu-va-giai-thuat/giai-thuat-tim-kiem-tuyen-tinh.jsp) *(linear search)* giữa hai vùng 0x18000000 và 0x28000000, mỗi bước nhảy ~500MB, việc tìm kiếm này sẽ chiếm nhiều nhất 8 lần truy vấn oracle.
2. Thực hiện [*tìm kiếm nhị phân*](https://vietjack.com/cau-truc-du-lieu-va-giai-thuat/giai-thuat-tim-kiem-nhi-phan.jsp) *(binary search)* giữa các địa chỉ tìm được ở bước một. Bước tìm kiếm này cũng chỉ chiếm một vài truy vấn thôi vì nó chạy trong thời gian logarithm.

Để bypass ASLR thì ít nhất 10 và không nhiều hơn 20 tin nhắn iMessage được gửi đi.

Tuy nhiên trong thực tế thì lỗ hổng bộ nhớ sẽ không giúp chúng ta thực hiện truy vấn một cách dễ dàng như thế này. Vì vậy ta cần phải có một phiên bản tổng quát hơn của thuật toán trên. Dù sao thì trong mọi trường hợp, để khai thác được lỗ hổng trước hết phải đáp ứng được một số kĩ thuật khai thác nhất định. Đây cũng là yêu cầu dành cho CVE-2018–8641.

Đầu tiên, bug trigger đã nhắc đến ở phần một của series này luôn làm crash hệ thống mặc dù có địa chỉ hợp lệ.

```objective-c
-[NSSharedKeyDictionary initWithCoder:coder] {
  self->_keyMap = [coder decodeObjectOfClass:[NSSharedKeySet class]
                         forKey:"NS.skkeyset"];
  // ... decode values etc.
}

-[NSSharedKeySet initWithCoder:coder] {
  self->_numKey = [coder decodeInt64ForKey:@"NS.numKey"];
  self->_rankTable = [coder decodeBytesForKey:@"NS.rankTable"];
  // ... copy more fields from the archive
  self->_subSharedKeySet = [coder 
                            decodeObjectOfClass:[NSSharedKeySet class]
                            forKey:@"NS.subskset"]];

  NSArray* keys = [coder decodeObjectOfClasses:[...] 
                         forKey:@"NS.keys"]];
  if (self->_numKey != [keys count]) {
    return fail(“Inconsistent archive);
  }
  self->_keys = calloc(self->_numKey, 8);
  // copy keys into _keys

  // Verify that all keys can be looked up
  for (id key in keys) {
    if ([self indexForKey:key] == -1) {
      NSMutableArray* allKeys = [NSMutableArray arrayWithArray:keys];
      [allKeys addObjectsFromArray:[self->_subSharedKeySet allKeys]];
      return [NSSharedKeySet keySetWithKeys:allKeys];
    }
  }
}
```

ObjC id được đọc từ một địa chỉ do người tấn công kiểm soát sau đó id này sẽ được so sánh với key đang được tìm kiếm (dòng 13 của phần `[NSSharedKeySet indexForKey:]`). Nếu như nó không tương đồng, việc lookup sẽ fail và `[NSSharedKeySet initWithCoder:]` sẽ cố tạo lại NSSharedKeySet từ đầu. Vì thế nên nó sẽ gọi `[NSSharedKeySet allKeys]` trong `subKeySet` của nó. Không may rằng, vì `subKeySet` chưa hoàn toàn được khởi tạo (bug có nhắc đến ở phần 1), phương thức `allKeys` sẽ crash ngay lập tức khi nó truy cập vào mảng `_keys` khi nó vẫn là nullptr (mảng rỗng, chưa được khởi tạo). Mặc dù vậy, ta vẫn có thể làm việc với chúng:

<img src="https://miro.medium.com/max/800/0*H5cwg78sSnpd3CGB.png" alt="Image for post" style="zoom:80%;" />

Trick ở đây là ta thêm vào cuối mảng một KeySet mới (SharedKeySet3), keyset này sẽ luôn có thể look up key thứ hai (“k2”). Tuy nhiên, do KeySet này đã trở thành subKeySet mới của `SharedKeySet1`, nên `SharedKeySet2` bắt buộc phải được unarchive (giải nén) bằng cách khác. Vấn đề này chỉ khả thi khi ta unarchive `SharedKeyDictionary` trước bằng cách thông qua mảng `_keys` của `SharedKeySet1`. Xui là whitelist của class này khi được dùng để unarchive `_keys` không hề xài đến NSDictionary. Dù vậy, class `__NSLocalizedString` (bản thân nó giống NSString) có một đoạn code giúp ta decode config dictionary của nó:

```
NSSet* classes = [NSSet setWithObjects:[NSDictionary class], ...];
NSDictionary* configDict = [coder decodeObjectOfClasses:classes 
                                  forKey:@"NS.configDict"]
```

Vì NSSharedKeyDictionary có thể được decode trong khi mảng `_keys` đang được giải nén bằng cách “gói gọn” bản thân nó vào __NSLocalizedString.

Bằng cách này, việc unarchive payload chỉ crash khi đáp ứng một trong hai yếu tố:

1. Nếu địa chỉ (trong trường hợp này `0x41414140` được xem như index của `_rankTable` được nhân lên 8) không đọc được (hay chưa được map lại).
2. Nếu việc gọi `[key isEqual:candidate]` với giá trị được được từ địa chỉ bị crash.

Nếu có giá trị là 0, `[key isEqual:]` sẽ không được gọi. Ngược lại, nếu giá trị không được set MSB, nó sẽ được xem như một con trỏ đến ObjC object và các phương thức sẽ được gọi dựa theo nó, chắc chắn việc này sẽ dẫn đến crash *trừ khi giá trị con trỏ là một ObjC object*. Cuối cùng, nếu giá trị đã được set MSB, nó sẽ dược xem như một con trỏ vùng nhớ (tagged pointer) và class của nó sẽ hợp lệ. Điều này xảy ra bằng cách XOR con trỏ vùng nhớ với *một giá trị random được mã hóa*, sau đó truyền index vào class table từ các bit cao hơn và sử dụng nó. Do giá trị XOR chung với tagged pointer bị mã hóa, ta không thể biết rằng giá trị nào khi truyền vào con trỏ sẽ làm crash. Tuy nhiên có một giải pháp khiến ta dù có truyền một địa chỉ sai vào tagged pointer thì cũng sẽ không gây crash: Chính là nhờ phương thức `[NSCFString isEqual:]`, nó được sử dụng khi key đang tham chiếu là một chuỗi:

```
if (a3 & 0x8000000000000000) {
  // Extract class index from tagged pointer
  v5 = ((a3 ^ objc_debug_taggedpointer_obfuscator) >> 60) & 7;
  if ( v5 == 7 )
    // Use extended class index in bits 52 - 60
    v5 = (((a3 ^ objc_debug_taggedpointer_obfuscator) >> 52) & 0xFF) + 8;// Check class index equals the one for NSString
  if ( v5 == 2 )
    // If yes, extract string content from lower bits and compare
    return _NSTaggedPointerStringEqualCFString(a3, self);// If not, just return false directly
  return 0;
}
```

Phương thức này khiến cho mọi giá trị có cờ MSB set được sử dụng như tham biến mà không tạo ra crash.

Kết hợp những thứ trên, oracle function đã trở nên khái quát hơn:

```
oracle(addr):
  if isMapped(addr) and 
     (isZero(*addr) or hasMSBSet(*addr) or pointsToObjCObject(*addr)):
    nocrash()
  else:
    crash()
```

Để sử dụng hàm oracle này, ta cần phải build một “profile” trên vùng shared cache trên máy nạn nhân (một vùng bitmap), giá trị ‘0’ có nghĩa việc truy cập sẽ crash và ‘1’ thì không. Vì vùng shared cache giống nhau hoàn toàn trên các thiết bị iPhone có hardware model giống nhau và cùng phiên bản iOS, việc này có thể được thực hiện bằng cách chạy một custom app trên một thiết bị tương đồng và scan toàn bộ vùng shared cache trong bộ nhớ. Trên thực tế, profile cũng nên hỗ trợ một trạng thái “dự phòng” để khi gặp hai trường hợp 0/1 trên bitmap. Nhưng để đơn giản hóa vấn đề, giải thích sau sẽ giả định profile chỉ bao gồm hai trạng thái chính.

Bitmap cho một profile gồm hai trạng thái như sau:

![Image for post](https://miro.medium.com/max/800/0*nQYTEoKDkRjcSqTP.png)

Bước kế tiếp là sử dụng hàm `oracle` dể tìm kiếm tuyến tính giữa vùng 0x18000000 và 0x28000000 cho đến khi không có crash nào xảy ra. Sau đó, tính toán các địa chỉ trong shared cache bằng cách lặp profile với pagesized step (page memory là vùng nhớ ảo). Trong thực tế, bước này sẽ cho kế quả từ 30000~40000 các `base_address` khác nhau (candidate).

Tiếp theo, một thuật toán tìm kiếm sẽ được sử dụng để nâng cao hiệu suất tìm ra địa chỉ chính xác mà chỉ sử dụng một lượng ít truy vấn oracle (mỗi truy vấn oracle sẽ mất khoảng 10s để tránh việc imagent bị crash quá nhanh). Ảnh sau sẽ biểu diễn shared cache map lẫn nhau bằng các `base_address`

![Image for post](https://miro.medium.com/max/1600/0*P25quU1fYgWdCN3u.png)

Mục đích hiện tại của chúng ta là tìm một địa chỉ mới bằng cái gửi truy vấn oracle. Địa chỉ mới này sẽ cho phép loại đi ít nhất nửa số `base_address` không hợp lệ mà ta đã tìm được (còn ~15000 đến 20000). Giờ hãy chú ý vào địa chỉ `0x19020c028` (màu xanh lá), nếu crash xuất hiện khi ta đang truy vấn oracle cho địa chỉ đó thì chỉ có `base_address` đầu tiên và cuối cùng được giữ lại, còn nếu nó không crash thì có ba `base_address` ở giữa sẽ được giữ lại.

*(Xem hình giải thích ở dưới để hiểu tại sao khi crash địa chỉ màu xanh lại loại được ba* `base_address` *ở giữa và ngược lại)*

![ Hai ô mình khoanh tròn là các chunk data mà 0x19020c028 tham chiếu đến](https://miro.medium.com/max/1600/1*vg3qIupTwHb7VooRR_0e6g.png)

Từ đó ta sẽ có được các `base_address` cần tìm, đồng thời việc này cũng đã cho biết tỉ lệ của việc crash (trong trường hợp này là 2/5). Ta suy ra được số lượng `base_address` còn lại (E) sau khi truy vấn oracle:
$$
E = 3/5 * 3 + 2/5 * 2 = 2.6
$$
Sau khi lặp liên tục, thuật toán sẽ chọn ra địa chỉ với giá trị (E) nhỏ nhất. Lý tưởng là khi các số 1 và 0 trong profile gần như cân bằng (thực tế thì có thể nhiều hoặc ít hơn) thì `base_address` với số (E) ít nhất sẽ được tìm ra trong vòng 2–5 phút.

Khi ứng dụng thuật toán này, ta sẽ gặp phải một vấn đề nhỏ về hiệu suất vì khi thực hiện full search cho các `base_address`, số operation phải chạy được tính theo công thức: `shared_cache_size / 8 * num_candidates`. Việc này có thể khiến số operation lên đến nghìn tỉ (10¹²). Mặc dù vậy, trong thực tế thì chúng ta không test tuần tự từng địa chỉ mà chỉ cần test ngẫu nhiên 100 địa chỉ khác nhau mà thôi.

Một vấn đề khác xuất hiện khi ta sử dụng “profile-ba-trạng-thái” được nhắc đến ở trên thì thuật toán sẽ ngầm định các writable-memory-page (một vùng nhớ) vừa có thể crash vừa có thể không (50/50) vì ta không biết được giá trị nào sẽ nằm trong vùng nhớ đó vào thời điểm runtime. Tuy nhiên, một lần nữa, thì khả năng crash trong thực tế lại hoạt động bình thường vì vấn đề nhỏ này không ảnh hưởng đến độ chính xác của thuật toán trên.

Sau đây là pseudocode phiên bản hoàn thiện của thuật toán trên:

```pseudocode
candidates = [...]
while len(candidates) > 1:
  best_address = 0x0
  best_E = len(candidates)
  remaining_candidates_on_crash = None
  remaining_candidates_on_nocrash = Nonefor _ in range(0, 100):
    addr = random.randrange(minbase, maxbase, 8)
    crashset = []
    nocrashset = []
    for profile in candidates:
      if profile.addr_will_crash(addr):
        crashset.append(profile)
      if profile.addr_will_not_crash(addr):
        nocrashset.append(profile)crash_prob = len(crashset) / len(candidates)
    nocrash_prob = 1.0 - crash_prob 
    E = crash_prob * len(crashset) + nocrash_prob * len(nocrashset)
    if E < best_E:
      best_E = E
      best_address = addr
      remaining_candidates_on_crash = crashset
      remaining_candidates_on_nocrash = nocrashsetif oracle(best_address):
    candidates = remaining_candidates_on_nocrash
  else:
    candidates = remaining_candidates_on_crash
```

Output của exploit:

```python
> ./aslrbreaker.py
[!] Note: this exploit *deliberately* displays notifications to the target
[*] Trying to find a valid address...
[*] Testing address 0x180000000...
[*] Testing address 0x188000000...
[*] Testing address 0x190000000...
[*] Testing address 0x198000000...
[*] Testing address 0x1a0000000...
[*] Testing address 0x1a8000000...
[*] Testing address 0x1b0000000...
[*] Testing address 0x1b8000000...
[*] Testing address 0x1c0000000...
[+] 0x1c0000000 is valid!
[*] Have 34353 potential candidates for the dyld_shared_cache slide
[*] Shared cache is mapped somewhere between 0x181948000 and 0x203d64000
[*] Now determining exact base address of shared cache...
[*] 34353 candidates remaining...
[*] Best (approximated) address to probe is 0x1b12070d0 with a score of 17208.40
[*] 17906 candidates remaining...
[*] Best (approximated) address to probe is 0x1b8a353d8 with a score of 9144.48
[*] 9656 candidates remaining...
[*] Best (approximated) address to probe is 0x1bcb23de0 with a score of 5093.02
[*] 5104 candidates remaining...
[*] Best (approximated) address to probe is 0x1e172e3f8 with a score of 2754.83
[*] 2682 candidates remaining...
[*] Best (approximated) address to probe is 0x1b363c658 with a score of 1454.06
[*] 1728 candidates remaining...
[*] Best (approximated) address to probe is 0x1e0301200 with a score of 929.21
[*] 915 candidates remaining...
[*] Best (approximated) address to probe is 0x1b0c04368 with a score of 497.63
[*] 593 candidates remaining...
[*] Best (approximated) address to probe is 0x1e0263068 with a score of 319.15
[*] 326 candidates remaining...
[*] Best (approximated) address to probe is 0x1bec43868 with a score of 163.84
[*] 156 candidates remaining...
[*] Best (approximated) address to probe is 0x1c15ab0e8 with a score of 78.21
[*] 82 candidates remaining...
[*] Best (approximated) address to probe is 0x1c49efe90 with a score of 41.02
[*] 40 candidates remaining...
[*] Best (approximated) address to probe is 0x1befd60f8 with a score of 20.00
[*] 20 candidates remaining...
[*] Best (approximated) address to probe is 0x1c14089d0 with a score of 10.00
[*] 10 candidates remaining...
[*] Best (approximated) address to probe is 0x1c428d450 with a score of 5.00
[*] 5 candidates remaining...
[*] Best (approximated) address to probe is 0x1df0939f0 with a score of 2.60
[*] 2 candidates remaining...
[*] Best (approximated) address to probe is 0x1c3d255f8 with a score of 1.00
[+] Shared cache is mapped at 0x1bf2b4000
```

Gần như những oracle function có thể được xây dựng kể cả từ các lỗ hổng bộ nhớ khác nhau. Ví dụ như tất cả các lỗ hổng cho phép attacker làm hư hại hoặc giả mạo một ObjC object ([CVE-2019–8641](https://bugs.chromium.org/p/project-zero/issues/detail?id=1881), use-after-free [CVE-2019–8647](https://bugs.chromium.org/p/project-zero/issues/detail?id=1873) và [CVE-2019–8662](https://bugs.chromium.org/p/project-zero/issues/detail?id=1874)) cũng có thể sử dụng để xây dụng oracle function sau đó đưa vào ứng dụng như trên.

*(Trans: Phần này tác giả nói thêm về cách phương thức objc_release hoạt động và cách construct oracle function)*

Khi việc tham chiếu đến một ObjC object bị can thiệp, method [objc_release](https://github.com/opensource-apple/objc4/blob/cd5e62a5597ea7a31dccef089317abb3a661c154/runtime/NSObject.mm#L1505) sẽ được gọi với object trên làm tham biến, nó sẽ kiểm tra xem object có “phương thức release đặc biệt” nào không bằng cách kiểm tra một bit trong Class object của object được tham chiếu đó (Class tham chiếu này được gọi là “ISA” (Is-a) pointer). Nếu object không có “phương thức release đặc biệt nào” thì nó sẽ giảm inline refcount (biến đếm phần tử) của object xuống, nếu kết quả không bằng ‘0’ -> return. Ngược lại thì nó sẽ gọi destructor của object đó và memory chunk được giải phóng.

```objective-c
objc_release(id obj)
{
    if (!obj) return;
    if (obj->isTaggedPointer()) return;
    return obj->release();
}
```

Nếu Class pointer của một object có thể được thay đổi để trỏ vào vùng shared cache, và đồng thời inline refcount > 1, thì objc_release chỉ crash nếu “bit đặc biệt” được set tại một offset từ con trỏ đến giá trị. Bằng cách này, crash oracle có thể được reconstruct.

## Hỗ trợ các phiên bản iOS và Hardware Model khác nhau

Attacker có thể build shared cache profile cho tất cả các hardware model và iOS version nếu không biết rõ về thông tin trên thiết bị của đối tượng nhắm đến. Việc này có thể cho khoảng vài trăm ngàn `base_address` nhưng nhờ vào sự “kì diệu” của thuật toán logarithm, kể cả khi số lượng lên đến hàng ngàn thì attacker chỉ cần gửi vài chục tin nhắn để xác định ra `base_address` cần tìm với shared cache, model number, version một cách chính xác.

# Lưu ý về sự ồn ào (noisiness) của việc exploit

Một điểm yếu khác của cách tấn công này là việc gây chú ý của nó. Trong khi crash vùng imagent mấy chục lần thì người dụng cũng không biết gì, tuy nhiên, thiết bị sẽ gửi crash report cho Apple nếu option “Share iPhone Analytics” được bật. Vì vậy, kĩ thuật này trở nên không hữu dụng lắm, nhưng may mắn rằng iOS sẽ ngưng việc log lại sau khi bị crash 25 lần

```verilog
default 14:54:42.957547 +0200 ReportCrash Formulating report for corpse[597] imagent
default 14:54:42.977891 +0200 ReportCrash Report of type '109(<private>)' not saved because the limit of 25 logs has been reached
```

Ta có thể thử DoS (VD như thực hiện đệ quy nhiều lần dẫn đến tràn bộ đệm) để crash imagent 25 lần trước khi thực hiện exploit để không bị iOS log lại. *Tuy nhiên cách này vẫn chưa được xác thực trong thực tế*.

# Vấn đề với Automatic Delivery Receipts

Như đã trình bày ở trên, việc bypass ASLR bằng cách tạo một side channel để gửi tin nhắn tự động từ thiết bị đến máy chủ của người tấn công là khả thi. Để vá lỗi này không chỉ đơn giản như kiểu: Gửi delivery receipt **trước** khi parse bất cứ thứ gì phức tạp, từ đó message vẫn được gửi đi mặc kệ việc payload có làm crash gì đi nữa. Tuy nhiên. cách này không bao quát được toàn bộ, kiểu tấn công sau đây vẫn có thể hoạt động như thường:

1. Gửi "oracle" message (cái mà gây crash) 2-3 lần liên tiếp
2. Gửi một tin nhắn "bình thường" (không gây crash)
3. Canh thời gian khi delivery receipt được gửi tới. Nếu nó lâu hơn một vài giây thì chắc hẳn vùng `imagent` đã crash từ bước 1 và việc gửi bị delay bởi launchd

Cách tấn công này đã ép buộc launchd phải restart và delay. Phương thức này cũng có thể tồn tại trên một số platform khác. Chính vì thế, ta hiểu được rằng bất kì tin nhắn tự động nào được gửi từ phía victim thì luôn có thể bị khai thác như cách trên. Cách giải quyết tốt nhất bây giờ là không cho bất kì tin nhắn nào được gửi một cách tự động cả, hoặc ít nhất phải có tác động của người dùng.

Phần hai của series *Remote iPhone Exploitation* sẽ kết thúc tại đây. Hẹn gặp các bạn ở phần tiếp theo.