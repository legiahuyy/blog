---
title: Nhận thức về HTTPS/SSL miễn phí
author: Huy
date: 2019-11-22 9:55:22 +0800
tags: [network]
---

Đa phần mọi người cho rằng những website với biểu tượng “khóa xanh” thì *an toàn.* Liệu có thực sự là như vậy?

![Source: Google Image](https://miro.medium.com/max/1366/0*leHE8ZDdWp9PGVWP)

Đầu tiên mình sẽ giải thích về HTTPS/SSL/TLS một cách đơn giản nhất cho những người chưa biết.

- **SSL (Secure Sockets Layer)** có thể hiểu nôm na là một chuẩn (standard) về an ninh công nghệ đảm bảo cho sự kết nối giữa 2 máy chủ an toàn, bảo mật và riêng tư. Đồng thời ngăn cản kẻ xấu có thể đọc và thay đổi dữ liệu (các kiểu tấn công MiTM, etc).
- **HTTPS (Hyper Text Transfer Protocol Secure)** là một giao thức bảo mật, biểu tượng của HTTPS xuất hiện trên thanh địa chỉ khi website đang truy cập có hỗ trợ SSL.
- **TLS (Transport Layer Security)**, nó đơn giản là một phiên bản nâng cấp của SSL.

Khả năng website của bạn có xuất hiện ở trang 1 khi tìm kiếm bằng Google hay không cũng phụ thuộc vào việc hỗ trợ SSL của website. Hay nói cách khác, mức độ tin tưởng của người dùng khi ghé web của bạn sẽ tăng khi có sự xuất hiện của “https://”.

Chính vì lí do này, các SEO-er đã không ngừng tìm cách để website của mình có chứng chỉ SSL. Dẫn đến việc, [Let’s Encrypt](https://letsencrypt.org/), [SFF](https://www.sslforfree.com/), [ZeroSSL](https://zerossl.com/) — các dịch vụ cung cấp SSL/TLS (CA) miễn phí ra đời => nhà nhà ai cũng có “khóa xanh”.

> Thế thì càng tốt, chẳng phải nếu website nào cũng hỗ trợ SSL thì toàn bộ kết nối của chúng ta sẽ được mã hóa và an toàn ư?

Ummmm… hoặc là không?

# SSL free cho tất cả mọi người!

Kể cả mấy trang phishing, lừa đảo luôn.

<img src="https://miro.medium.com/max/961/0*0d2CoX4VcCyRB6l0" alt="Source: Google Image" style="zoom: 67%;" />

Các dịch vụ cung cấp SSL miễn phí trên đã một phần tiếp tay cho những trang web “đáng tin cậy” như thế này trở nên đầy rẫy. Kẻ xấu đã đánh vào cái quan niệm “có https là an toàn” của hầu hết mọi người và thu thập những thông tin nhạy cảm như mật khẩu, số tài khoản ngân hàng.

Cái cần hiểu ở đây chính là SSL chỉ mã hóa thông tin được truyền đi để kẻ xấu không đọc/sửa được thông tin **trên đường truyền**. Nếu ta nhập dữ liệu và gửi đi đến máy chủ của tin tặc thì chúng vẫn có thể đọc được bình thường.

> Nhưng nếu là như vậy thì các dịch vụ cung cấp SSL đó vẫn có thể thu hồi lại chứng chỉ của mấy trang lừa đảo đó mà.

Đúng, tuy nhiên trước khi CA thu hồi lại chứng chỉ thì đã có bao nhiêu người đã bị lừa? Và chưa kể kẻ xấu vẫn có thể đăng kí lại chứng chỉ dưới một website khác, một tài khoản “sạch” khác.

## Mức độ mã hóa, key management và “mối nguy tiềm tàng” của free SSL dành cho các doanh nghiệp và người dùng

> The heart of encryption is the encryption key.

Giờ hãy tưởng tượng nhé, bạn có một ổ khóa siêu cứng có thể chịu được 1000°C, đạn bắn. NHƯNG điều gì sẽ xảy ra khi tên trộm có trong túi chiếc chìa khóa? Mối liên hệ ở đây là cái mức mã hóa dữ liệu (cipher strength) của chứng chỉ SSL mà website bạn đang sử dụng. Nếu kẻ xấu có được “private key” dùng để tạo ra chứng chỉ SSL đó thì dù cipher của ta có mạnh đến mấy thì cũng thành …giấy!

Có hàng loạt những dịch vụ cung cấp SSL trên Internet, mỗi dịch vụ có một hệ thống quản lí key riêng biệt, “Key Management System” hay KMS. Những dịch vụ này gửi những key này đến những nhà cung cấp hệ điều hành như Microsoft, Apple, Google để có được thứ gọi là “root certificates” — dùng để xác nhận chứng chỉ của họ.

<img src="https://miro.medium.com/max/1115/1*QTwFlLHTdnG8UzYEmhonUg.png" alt="Image for post" style="zoom:80%;" />

Lấy **Let’s Encrypt** làm ví dụ, càng nhiều website sử dụng dịch vụ của Let’s Encrypt, mối nguy hiểm càng lớn. Bởi vì nếu KMS của dịch vụ này bị lộ ra ngoài thì hàng ngàn website sử dụng SSL của họ sẽ bị ảnh hưởng. Khi chuyện xảy ra như thế, điều mà dịch vụ này sẽ làm đó là thu hồi lại toàn bộ SSL và tạo KMS mới, nếu người quản lí website không để ý đến việc SSL đã bị thu hồi thì toàn bộ thông tin người dùng lúc này sẽ trở nên “trần truồng”. Đây cũng là lí do dịch vụ tín dụng Equifax không hề hay biết mình đã bị hack trong vài tháng!

Hãy nghĩ như thế này, bạn là “người trung gian” giữa một website và người dùng của họ. Bạn có thể capture lại toàn bộ thông tin trao đổi giữa hai đối tượng đó, “private key” của SSL website đó đang sử dụng cũng đã nằm trong tay bạn và giờ bạn có thể decrypt lượng dữ liệu đã được mã hóa một cách dễ dàng. Điều tệ nhất là bạn với vai trò “người trung gian” này rất khó để bị phát hiện vì bạn không cần phải hack vào website cũng như cài malware vào thiết bị của người dùng. Kiểu tấn công này còn được gọi là MiTM (Man-in-the-Middle).

# Vậy ta xử lí như thế nào?

Ngày nay, mọi người có xu hướng kinh doanh, mua bán online thì những chủ kinh doanh lại muốn xây dựng website của mình sao cho đẹp, và thu hút nên họ mới tìm đến những giải pháp bảo mật “nhanh và gọn”. Nhưng thật lòng mà nói thì không có giải pháp nào “nhanh và gọn” cho bảo mật cả.

Thế những doanh nghiệp nhỏ, startup nên làm gì?

Đầu tiên: Gạt bỏ trong suy nghĩ “xài đồ free” đi. Những dịch vụ cung cấp SSL phổ biến có các công cụ mà bạn có thể sử dụng để đăng kí SSL cho website của mình một cách chính xác và bài bản. Những quy trình này cũng không hề rườm rà một cách vô lí. Đồng thời chú ý đến các điều khoản dịch vụ mà họ đưa ra nữa nhé.

Thứ hai: Nếu bạn sử dụng các dịch vụ hosting để host website của bạn thì hãy đọc kĩ nội dung thỏa thuận để xem ai là người chịu trách nhiệm quản lí bảo mật website của bạn vì có thể bạn sẽ phải nhờ họ gửi “yêu cầu đăng kí chứng chỉ” (CSR). Các dịch vụ cung cấp SSL sẽ cần CSR để tạo chứng chỉ.

Thứ ba: Chỉ đăng kí SSL với những dịch vụ cung cấp uy tín.

## Đối với người dùng

- Cẩn thận kể cả khi website bạn đang truy cập có “https”
- Mình khuyến khích các bạn sử dụng một trong hai extension/addon này: [HTTPS Everywhere](https://www.eff.org/https-everywhere) hoặc [ForceTLS](http://forcetls.sidstamm.com/).

Mình xin dừng ở đây, hẹn mọi người ở những bài kế tiếp nhé 💗