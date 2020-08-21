---
title: Xây dựng UPnP Botnet
author: Huy
date: 2019-11-22 9:55:22 +0800
categories: [Technical]
tags: [network, reverse engineer]
---

Cách tạo botnet 100% anonymity qua hàng loạt các thiết bị IoT bằng cách “sử dụng” mô hình giao thức UPnP.

------

# Inception Framework

Mới vừa rồi mình có đọc một [bài báo của Symantec](https://legiahuyy.blogspot.com/2020/08/xay-dung-upnp-botnet.html#) hồi tháng 3/2018 về nhóm hacker có tên là Espionage đã sử dụng lỗ hổng bảo mật trong UPnP để ẩn mình. Đồng thời, Symantec cũng nói rằng là không cần phải inject malware vào router vẫn có thể khai thác được lỗ hổng này.

# UPnP là gì?

Trước khi đào sâu vào vấn đề thì mình sẽ giải thích cho các bạn hiểu sơ khái niệm UPnP là gì.

UPnP là từ viết tắt của [Universal Plug and Play](https://en.wikipedia.org/wiki/Universal_Plug_and_Play), nó đơn giản là một mô hình protocol cho phép các thiết bị IoT tìm thấy nhau trong cùng một LAN và một số tính năng khác (như là chia sẻ dữ liệu, etc). UPnP là một mô hình protocol khá cũ rồi (phát triển và hoàn thiện từ 1990~2000) và phiên bản mới nhất hiện nay là UPnP Device Architecture 2.0 (2015).

UPnP chia thành 6 layers, nhưng ở đây mình sẽ nói 3 cái chính:

- **Discovery (Tìm kiếm)**: Hay còn gọi là [SSDP](https://en.wikipedia.org/wiki/Simple_Service_Discovery_Protocol) ( Simple Service Discovery Protocol), protocol này cho phép các thiết bị (có UPnP) tìm thấy nhau.
- **Description (Mô tả)**: Các thông tin về thiết bị được hiển thị dưới dạng XML bằng một remote URL, bằng cách này, thiết bị sẽ cung cấp cho mình tính tương thích và khả năng của nó.
- **Control (Điều khiển)**: Các *tín hiệu* điều khiển cũng sẽ được cung cấp dưới dạng XML thông qua SOAP protocol(kiểu kiểu như RPC nhưng không cần authentication).

Mình sẽ cho các bạn thấy các layers trên tương tác với nhau như thế nào qua ảnh minh họa.

![Image for post](https://miro.medium.com/proxy/1*pqBH9eLxfIx8IX2qvZz_AQ.png)

Để biết chi tiết hơn: [UPnP Doc 1.1](http://upnp.org/specs/arch/UPnP-arch-DeviceArchitecture-v1.1.pdf) | [2.0](https://openconnectivity.org/upnp-specs/UPnP-arch-DeviceArchitecture-v2.0-20200417.pdf).

# “Sử dụng” UPnP

Có rất nhiều cách để “sử dụng” các tính năng của UPnP, ở đây mình còn chưa nhắc tới [mấy cái CVEs](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=upnp) của UPnP nhé! Mà hầu hết mấy cái CVEs đó toàn là do misconfiguration hoặc do xài ..đồ cổ (poor implementations). Bài này mình sẽ tấn công kiểu **Open Forward** (cái này sẽ nói sau).

Bình thường thì UPnP sẽ hoạt động như thế này trong local:

![Image for post](https://miro.medium.com/proxy/1*nBv3IA5dgKJLiaAEJSu2eg.png)

SSDP sẽ mở port 1900/UDP đến IPv4 Local Scope multicast (ở đây là 239.255.255.250 hay ff0X::c), nó sẽ gửi một HTTPU (HTTP over UDP) packet `M-SEARCH`.

![Image for post](https://miro.medium.com/proxy/1*6tf7GvRIAMsKhOrf40k30g.png)

Bằng cái packet `M-SEARCH` này, nếu mình gửi nó đến một thiết bị UPnP có lỗ hổng (gọi là UPnP-V cho nhanh) thì nó sẽ reply lại mặc dù protocol là local-only! Do đó, mình có thể sử dụng router như một proxy.

![Image for post](https://miro.medium.com/proxy/1*-9SmUUqvuBc5OnGjC5vnMQ.png)

Lúc này mình sẽ nhận được response từ UPnP-V như sau:

![Image for post](https://miro.medium.com/proxy/1*52O8SF6mMaOvwQUBn2wRCA.png)

M-SEARCH response từ UPnP-V có chứa một dòng HTTP-Header là`Location` trỏ đến Device Description.

![Image for post](https://miro.medium.com/proxy/1*z2OhfPR1EWFs9uD4JzI-JA.png)

Để ý cái URL có chứa địa chỉ IP, từ đây mình có thể access được web server qua WAN khi truy cập vào cái địa chỉ public IP. Sau đó mình sẽ có được một cái doc ( SCPD — Service Control Protocol Document) như sau:

![Image for post](https://miro.medium.com/proxy/1*Jv0axVUpyx8LFoTUVdsntw.png)

Cái XML document này sẽ cho mình biết *một vài* *Actions set* & *State Variables* mà một *Service* thực thi.

Căn bản thì đây là nơi mà mình sẽ biết được các thông tin của UPnP-V. Ngoài ra cái UPnP XML còn cho mình `ControlURL` của mỗi service, hay gọi cách khác là SOAP endpoints dùng để *nói chuyện* với mấy cái service (kiểu như gửi một cái GET/POST để trigger action của service ấy).

Cái service mà mình thấy thú vị nhất trong đây chính là cái **WANIPConnection** service.

## WANIPConnection Service

Trích từ [UPnP standard](http://upnp.org/specs/gw/UPnP-gw-WANIPConnection-v2-Service.pdf):

> This service-type enables a UPnP control point to configure and control IP connections on the WAN interface of a UPnP compliant InternetGatewayDevice . Any type of WAN interface (e.g., DSL or cable) that can support an IP connection can use this service.
> […]
> An instance of a WANIPConnection service is activated (refer to the state variable table) for each actual Internet connection instance on a WANConnectionDevice. WANIPConnection service provides IP-level connectivity with an ISP for networked clients on the LAN.

Nói ngắn gọn thì cái service này là **UPnP NAT Toolbox.** Đọc là thấy có mùi rồi ;)

Tìm tiếp trong cái document vừa rồi thì mình sẽ thấy một cái function có tên là `AddPortMapping()` . Cái function này dùng để yêu cầu router chuyển hướng TCP/IP traffic đến một specific host/port trong LAN.

![Image for post](https://miro.medium.com/proxy/1*Zzncn0FSVZHTRVVmR_qBVw.png)

Ok, tiếp theo mình sẽ cho các bạn xem cách ta "sử dụng" hàm này như thế nào.

## The Open Forward Attack

Yep, kiểu tấn công này sẽ gọi UPnP SOAP functions từ WAN interface mà không cần authentication. Bằng function `AddPortMapping()` mình có thể:

1. Access một thiết bị sau NAT.
2. Access một thiết bị qua router.

Ngoài ra, đã có người sử dụng cái trick (1) để access Windows SMB port rồi exploit lỗ hổng EternalBlue.

Nhưng ở trong bài này mình sẽ chỉ nói về option (2):

![Image for post](https://miro.medium.com/proxy/1*8rCL34LVJs68QR5KbOoFsQ.png)

Kiểu tấn công này khá an toàn, “sạch sẽ”, chỉ cần “yêu cầu” thì router sẽ add cho mình một port mapping với đầy đủ các thông số. Sau đó, UPnP daemon sẽ “tặng” cho mình `iptables process` theo param mà mình đã chọn mà không thèm ..kiểm tra luôn! Do đó mình có thể chọn bất cứ public IP nào mà mình thích.

![Image for post](https://miro.medium.com/proxy/1*6-iIl7B4BdueHvfiFpXDxg.png)

Cứ tiếp tục lặp lại khoảng vài ba lần như thế là ổn.

# Conclusion

Đây chỉ là một trong rất nhiều cách tấn công khác mà UPnP *cho phép “sử dụng”.*

Tuy nhiên, cách này vẫn để lại “dấu vết” nhưng hầu hết là logs sẽ được lưu lại trên database của ISP thôi nên chắc sẽ không sao :/ (maybe).