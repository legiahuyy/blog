---
title: "Project Zero: Remote iPhone Exploitation [P1]"
author: Huy
date: 2020-01-08 4:15:52 +0800
categories: [Series, "Project Zero"]
tags: [translate, reverse engineer, 0day]
---

Bài viết này được mình dịch từ blog của team [Google Project Zero](https://googleprojectzero.blogspot.com/2020/01/remote-iphone-exploitation-part-1.html).

**Lưu ý:** Bài viết gốc của tác giả có rất nhiều những từ chuyên ngành dịch sang tiếng Việt nghe rất …oải và không trọn vẹn nghĩa nên mình sẽ để nguyên. Ngoài ra mình cũng có thêm một số giải thích để giúp bạn đọc hiểu rõ hơn về nghiên cứu của tác giả cũng như các từ chuyên môn trong bài. Enjoy!

# Lời mở đầu

Đây là blog post đầu trong series *Remote iPhone Exploitation.* Trong series này, chúng ta sẽ đi chi tiết về lỗ hổng trên iMessage có thể bị khai thác từ xa mà không cần user phải “đụng chân đụng tay”. Lỗ hổng này tồn tại ở phiên bản iOS 12.4 (hiện đã được vá trong iOS 12.4.1 hồi tháng 8/2019). Bạn đọc có thể tìm hiểu thêm [*tại đây*](https://media.ccc.de/v/36c3-10497-messenger_hacking_remotely_compromising_an_iphone_through_imessage)*.*

Phần một sẽ cung cấp cho chúng ta thông tin về lỗ hổng, phần tiếp theo sẽ nói về kỹ thuật bypass ASLR (**A**ddress **S**pace **L**ayout **R**andomization) và phần cuối cùng sẽ giải thích cách thực hiện RCE (**R**emote **C**ode **E**xecution — thực thi mã lệnh từ xa).

Với việc khai thác lỗ hổng trên, người tấn công chỉ cần có được Apple ID (số điện thoại hoặc email) thì sẽ kiểm soát được hoàn toàn thiết bị iOS của nạn nhân. Kế tiếp, kẻ xấu có thể can thiệp vào toàn bộ file, mật khẩu, mã xác minh, tin nhắn (SMS/etc), email và những thông tin khác. Chúng cũng có thể bật/tắt microphone và camera của thiết bị mà người dùng không hề hay biết vì không có bất cứ thông báo nào xuất hiện cả. Chỉ bằng cách khai thác một lỗ hổng duy nhất — [CVE-2019–8641](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-8641), ta có thể bypass ASLR rồi thực thi mã ngầm bên ngoài [sandbox](https://genk.vn/may-tinh/tim-hieu-ve-sandbox-hop-cat-bao-ve-an-toan-cho-may-tinh-cua-ban-20131029225422614.chn) trên thiết bị.

Bài viết này, đồng thời cũng là để giải đáp cho câu hỏi:

> Chỉ với một lỗ hổng về bộ nhớ, liệu ta có được RCE trên thiết bị iOS mà không cần khai thác thêm bất kì lỗ hổng nào hay không?

Câu trả lời là có, và giờ ta sẽ chứng minh.

## CVE-2019–8641

Lỗ hổng được tìm thấy trong một dự án tìm lỗ hổng chung với Natalie Silvanovich và đã [báo cáo cho Apple vào 29/7/2019](https://bugs.chromium.org/p/project-zero/issues/detail?id=1917), tiếp đó là PoC (Proof-of-Concept — nói đơn giản là chứng minh tính khả thi của việc khai thác) được gửi đi vào 9/8/2019. Lỗ hổng được Apple “vá tạm” bằng cách [chặn truy cập đến đoạn code có lỗ hổng](https://twitter.com/5aelo/status/1172534071332917248), sau đó được sửa hoàn toàn trên iOS 13.2.

Dành cho những ai muốn nghiên cứu, đây là [PoC trên iOS 12.4](https://bugs.chromium.org/p/project-zero/issues/detail?id=1917#c6) của iPhone XS.

# Mô hình hoạt động của iMessage

Những tin nhắn iMessage trước khi dến được với người dùng thì nó phải đi qua nhiều những service (dịch vụ) và framework. Main service xử lý những tin nhắn iMessage trên iOS 12.4 mà không cần sự tương tác của người dùng được biểu diễn như sau:

<img src="https://miro.medium.com/max/800/0*_frn4sUkDCN8rQes.png" alt="Các ô được khoanh đỏ là những tiến trình chạy trong sandbox của iOS" style="zoom:80%;" />

Những đối tượng bên ngoài sandbox mà ta có thể khai thác bao gồm [NSKeyedUnarchiver API](https://developer.apple.com/documentation/foundation/nskeyedunarchiver?language=objc) và data format của iMessage. Sau khi tìm hiểu, lỗ hổng trong NSKeyedUnarchiver API có thể được “kích hoạt” bằng hai cách:

- Trong tiến trình **imagent**.
- Ngoài sandbox, trong phần **SpringBoard** (tiến trình quản lí giao diện iOS, bao gồm màn hình khóa).

Cả hai cách này đều có điểm có lợi và bất lợi cho việc khai thác. Ví dụ, trong khi SpringBoard có thể chạy bên ngoài sandbox nhưng nó cũng sẽ hiện thông báo “respring” trên thiết bị việc ta can thiệp làm hệ thống bị crash (ai vọc jaibreak nhiều sẽ biết). Imagent thì không như thế nhưng nó lại nằm trong sandbox — mọi thay đổi sẽ bị xóa khi người dùng restart.

Trong bản vá iOS 13, việc decode data của NSKeyedUnarchiver không còn xảy ra trên SpringBoard nữa mà là trong sandbox IMDPersistenceAgent.

## Exploit iMessage

Để có thể gửi exploit qua iMessage, ta phải có khả năng gửi tin nhắn tùy chỉnh (custom message) đến mục tiêu. Điều này đòi hỏi việc phải tương tác với server của Apple và xử lý E2E (end-to-end) encryption của iMessage. Tuy nhiên, có một cách đơn giản hơn để làm điều này đó là “tái sử dụng” code xử lí trong **imagent** của iMessage. Công cụ như [frida](https://frida.re/) có thể giúp ta gửi đoạn tin nhắn custom đó trên MacOS như sau:

1. Build một payload (để “kích hoạt” bug của NSKeyedUnarchiver) và lưu nó vào disk.
2. Sử dụng AppleScript để ra hiệu cho [Messages.app](http://messages.app/) (một dịch vụ giả lập iMessage mà hình như đã bị gỡ xuống) gửi một tin nhắn bất kì (VD: “REPLACEME”) đến mục tiêu.
3. Hook **imagent** bằng frida và thay thế tin nhắn được gửi đi bằng payload vừa tạo.

Cũng bằng cách này ta có thể nhận tin nhắn bằng cách sử dụng frida để hook hàm *receiver* trong **imagent**.

Đoạn mã dưới đây là ví dụ, dòng tin nhắn iMessage (encode [binary plist](https://en.wikipedia.org/wiki/Property_list#macOS)) sẽ được gửi đến người nhận với nội dung “REPLACEME”:

```objc
{
    gid = "008412B9-A4F7-4B96-96C3-70C4276CB2BE";
    gv = 8;
    p =     (
        "mailto:sender@foo.bar",
        "mailto:receiver@foo.bar"
    );
    pv = 0;
    r = "6401430E-CDD3-4BC7-A377-7611706B431F";
    t = "REPLACEME";
    v = 1;
    x = "<html><body>REPLACEME</body></html>";
}
```

Công cụ Frida sẽ chỉnh sửa lại tin nhắn trước khi nó được đánh số, mã hóa và gửi đến server của Apple:

```objc
{
    gid = "008412B9-A4F7-4B96-96C3-70C4276CB2BE";
    gv = 8;
    p =     (
        "mailto:sender@foo.bar",
        "mailto:receiver@foo.bar”
    );
    pv = 0;
    r = "6401430E-CDD3-4BC7-A377-7611706B431F";
    t = "REPLACEME";
    v = 1;	
    x = "<html><body>REPLACEME</body></html>";
    ati = <content of /private/var/tmp/com.apple.messages/payload>;
}
```

Sau khi gửi, đoạn tin nhắn trên sẽ khiến cho tiến trình **imagent** của người nhận deocde dữ liệu trong *ati* bằng cách sử dụng NSKeyedUnarchiver API.

Tiếp theo, chúng ta sẽ sử dụng CVE-2019–8641 để tiến hành khai thác. Để hiểu rõ hơn về bug này, ta cần phải xem format serial của NSKeyedUnarchiver. Dưới đây là một archive đơn giản chứa serial NSSharedKeyDictionary (NSKeyedUnarchiver đã encode đối tượng trong graph thành *plist*):

```javascript
{
  "$archiver" => "NSKeyedArchiver"
  # The objects contained in the archive are stored in this array
  # and can be referenced during decoding using their index
  "$objects" => [
    # Index 0 always contains the nil value
    0 => "$null"
    # The serialized NSSharedKeyDictionary
    1 => {
      "$class" => <CFKeyedArchiverUID>{value = 7}
      "NS.count" => 0
      "NS.sideDic" => <CFKeyedArchiverUID>{value = 0}
      "NS.skkeyset" => <CFKeyedArchiverUID>{value = 2}
    }
    # The NSSharedKeySet associated with the dictionary    
    2 => {
      "$class" => <CFKeyedArchiverUID>{value = 6}
      "NS.algorithmType" => 1
      "NS.factor" => 3
      "NS.g" => <00>
      "NS.keys" => <CFKeyedArchiverUID>{value = 3}
      "NS.M" => 6
      "NS.numKey" => 1
      "NS.rankTable" => <00000000 0001>
      "NS.seed0" => 361949685
      "NS.seed1" => 2328087422
      "NS.select" => 0
      "NS.subskset" => <CFKeyedArchiverUID>{value = 0}
    }
    # The keys of the NSSharedKeySet 
    3 => {
      "$class" => <CFKeyedArchiverUID>{value = 5}
      "NS.objects" => [
        0 => <CFKeyedArchiverUID>{value = 4}
      ]
    }
    # The value of the first (and only) key
    4 => "the_key"
    # ObjC classes are stored in this format
    5 => {
      "$classes" => [
        0 => "NSArray"
        1 => "NSObject"
      ]
      "$classname" => "NSArray"
    }
    6 => {
      "$classes" => [
        0 => "NSSharedKeySet"
        1 => "NSObject"
      ]
      "$classname" => "NSSharedKeySet"
    }
    7 => {
      "$classes" => [
        0 => "NSSharedKeyDictionary"
        1 => "NSMutableDictionary"
        2 => "NSDictionary"
        3 => "NSObject"
      ]
      "$classname" => "NSSharedKeyDictionary"
    }
  ]
  # A reference to the root object in the archive
  "$top" => {
    "root" => <CFKeyedArchiverUID>{value = 1}
  }
  "$version" => 100000
}
```

Điều đáng chú ý ở đây là kiểu format này hỗ trợ tham chiếu đến các object bên trong list bằng các UID từ CFKeyedArchiverUID.

Trong khi đang unarchive, NSKeyedUnarchiver sẽ liên tục map lại các UID cho những object đó. Hơn nữa, việc tham chiếu được NSKeyedUnarchiver map lại trước khi phương thức initWithCoder được thực thi. Vì vậy nên ta có thể decode lần lượt các đối tượng đang được unarchive ngay trong callstack.

Từ đó ta biết được rằng object đầu tiên có thể *chưa được khởi tạo hoàn toàn* khi nó được tham chiếu dẫn đến sự tồn tại lỗ hổng CVE-2019-8641.

Pseudo-code (Object-C) của initWithCoder trong hai class NSSharedKeyDictionary và NSSharedKeySet như sau:

```pseudocode
[obj doXWith:y and:z];
obj->doX(y,z);
```

Nếu bạn không biết Object-C, có thể tham khảo bằng C++:

```c++
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

Ở dòng 20, *indexForKey* trong vòng lặp for đã sử dụng hash của một key để index vào *_rankTable* và sử dụng kết quả như một index vào *_keys*. Nó tìm tất cả các key trong subSharedKeySet cho đến khi tìm ra key đúng:

```c++
-[NSSharedKeySet indexForKey:] {
  NSSharedKeySet* current = self;
  uint32_t prevLength = 0;
  while (current) {
    // Compute a hash from the key and other internal values of    
    // the KeySet. Convert the hash to an index and ensure that it 
    // is within the bounds of rankTable
    uint32_t rankTableIndex = ...;
    uint32_t index = self->_rankTable[rankTableIndex];
    if (index < self->_numKey) {
      id candidate = self->_keys[index];
      if (candidate != nil) {
        if ([key isEqual:candidate]) {
          return prevLength + index;
        }
      }
    prevLength += self->_numKey;
    current = self->_subSharedKeySet;
  }
  return -1;
}
```

Với mớ logic trên, object graph sau đây sẽ chỉ rõ hơn về lỗi tràn bộ nhớ:

<img src="https://miro.medium.com/max/800/0*sCqKzBg2rcN6FJIp.png" alt="Image for post" style="zoom:80%;" />

Đây là những gì xảy ra trong quá trình unarchive:

1. Sau khi NSSharedKeyDictionary được unarchive, nó sẽ tự unarchive SharedKeySet của nó.
2. Phương thức `initWithCoder` của ShareKeySet1 được gọi khi subSharedKeySet của keyset này được unarchive. Tại thời điểm này:
   \+ `_numKeys` hoàn toàn nằm trong tầm kiểm soát của người tấn công mà không có sự kiểm tra nào (kiểm tra chỉ xảy ra sau khi *_keys* đã được unarchive)
   \+ `_rankTable` cũng được kiểm soát
   \+ `_keys` vẫn là một mảng rỗng (nullptr)
3. SharedKeySet2 đã được unarchive xong. subSharedKeySet của nó hiện tại đang tham chiếu đến SharedKeySet1 (keyset này vẫn chưa unarchive xong). Cuối cùng, nó sẽ gọi indexForKey cho mọi key trong mảng _keys (dòng 20)
4. rankTable của SharedKeySet2 đều có giá trị ‘0’ nên chỉ có key đầu tiên mới có thể tự map chính nó (xem dòng 8 đến 15 trong indexForKey). Từ key thứ 2 nó sẽ tham chiếu đến SharedKeySet1. Tại đây, vì _numKey và _rankTable đã được kiểm soát sẽ dẫn đến việc code bị crash.

Callstack tại thời điểm crash như sau:

Đây là những gì xảy ra trong quá trình unarchive:

1. Sau khi NSSharedKeyDictionary được unarchive, nó sẽ tự unarchive SharedKeySet của nó.
2. Phương thức `initWithCoder` của ShareKeySet1 được gọi khi subSharedKeySet của keyset này được unarchive. Tại thời điểm này:
   \+ `_numKeys` hoàn toàn nằm trong tầm kiểm soát của người tấn công mà không có sự kiểm tra nào (kiểm tra chỉ xảy ra sau khi *_keys* đã được unarchive)
   \+ `_rankTable` cũng được kiểm soát
   \+ `_keys` vẫn là một mảng rỗng (nullptr)
3. SharedKeySet2 đã được unarchive xong. subSharedKeySet của nó hiện tại đang tham chiếu đến SharedKeySet1 (keyset này vẫn chưa unarchive xong). Cuối cùng, nó sẽ gọi indexForKey cho mọi key trong mảng _keys (dòng 20)
4. rankTable của SharedKeySet2 đều có giá trị ‘0’ nên chỉ có key đầu tiên mới có thể tự map chính nó (xem dòng 8 đến 15 trong indexForKey). Từ key thứ 2 nó sẽ tham chiếu đến SharedKeySet1. Tại đây, vì _numKey và _rankTable đã được kiểm soát sẽ dẫn đến việc code bị crash.

Callstack tại thời điểm crash như sau:

<img src="https://miro.medium.com/max/800/0*z2e8N5jKrWFq8a3z.png" alt="Image for post" style="zoom:80%;" />

Do đó, một địa chỉ bất kì (trong trường hợp này là 0x41414140) sẽ được tham chiếu đến và được sử dụng như một con trỏ object (“id”). Đây chính là địa chỉ truy cập đến bug nhưng phải thỏa hai điều kiện:

1. Địa chỉ phải chia hết cho 8 (vì mảng `_keys `chứa giá trị kiểu con trỏ)
2. Địa chỉ phải ít hơn 32G vì index là một unsigned integer (4-byte)

May mắn là trên iOS, hầu hết (?) những thứ “đặc biệt” thường nằm dưới 0x800000000 (32G) nên ta có thể truy cập thoải mái.

Tóm lại, bài viết này cho ta thấy rằng một con trỏ rỗng (null pointer) lại trở thành một công cụ exploit nguy hiểm (ít nhất là trong trường hợp này).

Tuy nhiên tại thời điểm này, vẫn chưa có thông tin gì về địa chỉ của các tiến trình bên thiết bị của nạn nhân cả. Ta sẽ bàn về vấn đề này trong bài viết tiếp theo.