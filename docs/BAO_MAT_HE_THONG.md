# Báo cáo kỹ thuật: Xử lý sự cố máy chủ bị lợi dụng đào tiền mã hóa và gia cố bảo mật hệ thống (13/11/2026)

## 1. Vấn đề phát sinh và phạm vi ảnh hưởng
Hệ thống CatPlus Online Judge ghi nhận sự cố máy chủ bị xâm nhập và bị sử dụng trái phép để chạy phần mềm đào tiền mã hóa (crypto mining). Theo phản ánh từ các lớp trong quá trình sử dụng hệ thống, một số bài nộp bị treo hoặc không được chấm, khiến học viên phải chờ kết quả và việc theo dõi bài làm bị gián đoạn.

Phần mềm đào tiền mã hóa sử dụng tài nguyên tính toán của máy chủ. Khi chạy cùng các dịch vụ của CatPlus, tiến trình này có thể làm giảm tài nguyên dành cho ứng dụng và hệ chấm bài, dẫn đến xử lý chậm hoặc chậm trả kết quả. Vì vậy, việc xử lý sự cố cần bao gồm cả máy chủ, ứng dụng web và luồng chấm bài.

Tôi chia công việc thành ba nhóm:

- **Khôi phục máy chủ:** Cài lại hệ điều hành, kiểm tra ứng dụng và cấu hình trước khi đưa trở lại sử dụng.
- **Gia cố ứng dụng và đường truy cập:** Xử lý các điểm yếu của nền tảng phần mềm dùng để xây dựng website (framework), đồng thời tổ chức lại cách người dùng truy cập các dịch vụ.
- **Bảo vệ tài khoản và khôi phục chấm bài:** Thay mật khẩu, khóa truy cập có nguy cơ bị lộ; kiểm tra từng bài chưa có kết quả để xác định công việc cần xử lý lại.

Các phần dưới trình bày biện pháp xử lý, thay đổi trong cấu hình và quy trình kiểm tra. Trạng thái triển khai cùng những việc cần xác minh thêm được tổng hợp tại Mục 6.

## 2. Phân tích điểm yếu và hướng xử lý
Mã nguồn của trang quản lý nội bộ và trang học tập đều khai báo sử dụng Next.js `15.4.4`. Phiên bản này nằm trong nhóm bị ảnh hưởng bởi lỗ hổng thực thi mã từ xa trong React Server Components, được công bố dưới mã **CVE-2025-55182**, thường gọi là [React2Shell](https://github.com/vercel/next.js/security/advisories/GHSA-9qr9-h5gf-34mp).

Lỗ hổng thực thi mã từ xa cho phép đối tượng tấn công chạy mã trong môi trường ứng dụng, kể cả trước khi đăng nhập. Đây là điểm yếu cần ưu tiên xử lý ở cả hai website. Việc khắc phục cần cập nhật framework và các thư viện liên quan, tạo lại bản ứng dụng rồi triển khai lên máy chủ.

Hướng xử lý gồm ba phần:

- **Cập nhật ứng dụng:** Chọn phiên bản Next.js cùng các thư viện tương thích còn được hỗ trợ và đã vá các cảnh báo bảo mật hiện hành. Cập nhật cả tệp khai báo thư viện và tệp cố định phiên bản thư viện (lockfile), sau đó đóng gói lại hai website và kiểm tra phiên bản đang chạy.
- **Tổ chức lại đường truy cập:** Dùng cấu hình riêng cho máy chủ phục vụ người dùng thật (production). Trong cấu hình Docker này, cổng của các dịch vụ bên trong không được ánh xạ thành cổng truy cập trên VPS. Yêu cầu web đi qua Cloudflare Tunnel rồi đến Nginx, thành phần tiếp nhận và chuyển yêu cầu tới đúng dịch vụ.
- **Bảo vệ quyền truy cập:** Thay mật khẩu, khóa và token có nguy cơ bị lộ; kiểm tra quyền quản trị máy chủ và quyền của các tiến trình chấm bài.

Cập nhật framework và sử dụng Tunnel giải quyết hai vấn đề khác nhau: bản vá sửa lỗi trong ứng dụng, còn Tunnel tổ chức đường tiếp nhận lưu lượng từ bên ngoài. Hai lớp bảo vệ này cần được thực hiện cùng nhau.

## 3. Khôi phục máy chủ và gia cố đường truy cập
Tài liệu xử lý sau sự cố ghi nhận VPS đã được cài lại hệ điều hành. Bước này tạo lại môi trường chạy ứng dụng trên máy chủ. Các ứng dụng, cấu hình và thông tin xác thực được đưa trở lại máy cần được kiểm tra trong quá trình khôi phục.

Phần cấu hình triển khai đã được bổ sung theo ba hướng:

1. **Tách cấu hình máy chủ khỏi cấu hình phát triển:** Tệp `compose.server.yaml` được tạo riêng để triển khai hệ thống phục vụ người dùng. Máy cá nhân tiếp tục dùng cấu hình hiện có. Cách tách này giúp quản lý rõ các thiết lập dành cho từng môi trường và tránh mang nhầm thiết lập mở cổng của máy cá nhân lên VPS.
2. **Đưa yêu cầu web qua Cloudflare Tunnel và Nginx:** Cấu hình mới bỏ khai báo ánh xạ cổng dịch vụ Docker ra cổng trên VPS. Cloudflare Tunnel chuyển yêu cầu tới Nginx trong mạng Docker; Nginx tiếp tục chuyển yêu cầu đến website hoặc dịch vụ tương ứng. Các thành phần phía sau trao đổi qua những mạng đã khai báo trong cấu hình.
3. **Điều chỉnh chuyển tiếp yêu cầu và cookie đăng nhập:** Cấu hình Nginx dành cho máy chủ được đặt tại `infrastructure/gateway/nginx.server.conf`. Tệp này thiết lập thông tin chuyển tiếp cho luồng HTTPS qua Tunnel và ẩn thông tin phiên bản Nginx. Tuyến API học tập được bổ sung chính sách `private, no-store` để hạn chế việc lưu phản hồi vào bộ nhớ đệm. Hai website mặc định bật thuộc tính `Secure` cho cookie phiên trong cấu hình production; thuộc tính này yêu cầu trình duyệt gửi cookie qua HTTPS.

Các thay đổi trên làm rõ đường tiếp nhận yêu cầu, giảm các cổng dịch vụ được mở trực tiếp trên VPS và bổ sung cách bảo vệ phản hồi API cùng cookie đăng nhập. Việc kiểm tra cấu hình được trình bày tại Mục 5.

## 4. Bảo vệ thông tin xác thực và dịch vụ chấm bài
Mật khẩu hoặc khóa đã bị sao chép vẫn có thể được sử dụng sau khi máy chủ cài lại hệ điều hành. Vì vậy, quy trình xử lý sau sự cố yêu cầu thay các thông tin xác thực từng lưu trên VPS cũ và thu hồi giá trị cũ tại những dịch vụ đang sử dụng chúng.

Các nhóm cần thay gồm:

- **Mật khẩu và khóa dịch vụ:** Mật khẩu cơ sở dữ liệu cùng các khóa bảo mật của dịch vụ phía máy chủ.
- **Khóa ký thông tin đăng nhập:** Cặp khóa JWT dùng để tạo và kiểm tra token truy cập của người dùng.
- **Khóa giao tiếp giữa các dịch vụ:** Các khóa dùng để xác thực yêu cầu nội bộ, kết nối thành phần điều phối với các tiến trình chấm bài.
- **Thông tin kết nối hạ tầng:** Token Cloudflare Tunnel. Đồng thời, cần kiểm tra lại tài khoản quản trị, khóa SSH và quyền đăng nhập máy chủ.

Tệp cấu hình chứa mật khẩu hoặc khóa phải được lưu ngoài kho mã nguồn và chỉ cấp quyền đọc cho người cần sử dụng. Sau khi thay khóa, cần kiểm tra cả khả năng sử dụng khóa mới và việc khóa cũ đã hết hiệu lực.

Để khôi phục các bài nộp bị ảnh hưởng, các bước cần thực hiện là:

1. **Lập danh sách bài cần kiểm tra:** Đối chiếu dữ liệu bài nộp với thông tin từ hệ chấm. Phân biệt bài chưa được tiếp nhận, đang chờ xử lý và đã có kết quả nhưng giao diện chưa cập nhật đầy đủ.
2. **Kiểm tra hệ chấm trước khi gửi lại bài:** Xác nhận thành phần điều phối và các tiến trình chấm đang hoạt động, có thể tiếp nhận công việc.
3. **Xử lý lại theo trạng thái từng bài:** Giữ nguyên mã nguồn và lịch sử kết quả. Chấm lại những bài cần khôi phục theo cơ chế của hệ thống, tránh sửa trạng thái hoặc xóa dữ liệu bài nộp để giải phóng bài bị treo.
4. **Kiểm tra kết quả học viên nhận được:** Đối chiếu kết quả lưu trên máy chủ với trạng thái hiển thị ở tài khoản học viên để xác nhận bài đã được xử lý đầy đủ.

Riêng DMOJ là thành phần chạy mã nguồn bài nộp. Việc rà soát thành phần này cần tập trung vào quyền của tiến trình chấm, các tệp được phép truy cập và khả năng kết nối mạng. Đây là phần bảo vệ môi trường chạy bài nộp, bên cạnh việc bảo vệ đường truy cập web.

## 5. Phương pháp kiểm tra và kết quả ghi nhận
Theo tài liệu triển khai, cấu hình bảo mật sau chỉnh sửa đã được kiểm tra trên máy cá nhân. Các kết quả của đợt kiểm tra đó được ghi lại như sau:

- **Kiểm tra tệp cấu hình và danh sách dịch vụ:** Tệp YAML được kiểm tra cú pháp; Docker Compose được dùng để kiểm tra cấu hình với mật khẩu và khóa thử nghiệm. Kết quả đối chiếu ghi nhận cấu hình máy chủ giữ các dịch vụ và vùng lưu dữ liệu từ bản phát triển, bổ sung Cloudflare Tunnel và bỏ các khai báo ánh xạ cổng dịch vụ ra VPS.
- **Kiểm tra cách Nginx chuyển yêu cầu:** Nginx được chạy thử trên máy cá nhân với các dịch vụ mô phỏng đóng vai trò nhận và trả lời yêu cầu. Thử nghiệm kiểm tra yêu cầu được chuyển đến đúng nơi, thông tin tên miền và giao thức HTTPS được chuyển tiếp đúng, cùng chính sách bộ nhớ đệm của tuyến API học tập.
- **Kiểm tra việc phát hiện cấu hình còn thiếu:** Cấu hình máy chủ yêu cầu cung cấp các biến bắt buộc trước khi chạy, bao gồm những biến chứa mật khẩu và khóa. Cấu hình phát triển được giữ riêng và tiếp tục dùng theo quy trình hiện có.

Các kiểm tra này phục vụ việc chuẩn bị bộ cấu hình triển khai. Phần kiểm tra trên VPS cần tiếp tục xác nhận kết nối Tunnel, đăng nhập, cookie và hoạt động chấm bài theo các nội dung ở Mục 6.

## 6. Trạng thái xử lý và các việc cần xác minh thêm
Phần khôi phục và gia cố đã có trong tài liệu dự án gồm việc cài lại hệ điều hành VPS, bổ sung cấu hình production riêng qua Cloudflare Tunnel và hướng dẫn thay thông tin xác thực. Để hoàn tất việc xử lý sự cố, cần xác nhận thêm các nội dung sau:

- **Nguyên nhân và phạm vi xâm nhập:** Hồ sơ hiện có gồm mô tả sự cố và tài liệu xử lý. Cần bổ sung nhật ký truy cập web, Docker, tiến trình và kết nối mạng gốc để xác định thời điểm tấn công, cách đối tượng xâm nhập và phần mềm đào tiền mã hóa đã chạy. Quyền quản trị máy chủ mà đối tượng có được và việc truy cập dữ liệu người dùng cũng cần được xác minh bằng các dấu vết này. React2Shell hiện là hướng điều tra dựa trên điểm yếu của mã nguồn.
- **Bản vá của hai website:** Mã nguồn được kiểm tra vẫn khai báo Next.js `15.4.4`. Việc nâng phiên bản, đóng gói lại và xác nhận bản đã vá trong cả hai môi trường chạy ứng dụng Docker (container) là phần cần hoàn tất.
- **Triển khai và bảo vệ truy cập trên VPS:** Cần có kết quả xác nhận bộ cấu hình mới đã được áp dụng trên VPS, Tunnel hoạt động, người dùng đăng nhập bình thường và các mật khẩu, khóa cũ đã được thu hồi. Việc đóng gói và chạy thử toàn bộ hệ thống cần được kiểm tra riêng, bên cạnh các thử nghiệm cấu hình ở Mục 5.
- **Kết quả chấm bài sau khôi phục:** Cần có danh sách bài bị ảnh hưởng và kết quả xử lý từng bài để xác định số lượng, thời gian gián đoạn và trạng thái khôi phục. Hoạt động chấm bài được kiểm tra đến cả kết quả lưu trên máy chủ và kết quả hiển thị cho học viên.

Mục tiêu xử lý là khôi phục máy chủ, sửa điểm yếu của ứng dụng, bảo vệ lại quyền truy cập và đưa các bài nộp bị ảnh hưởng trở về luồng chấm bình thường. Kết quả kiểm tra từng phần là căn cứ để xác nhận tiến độ khắc phục và trạng thái hoạt động của hệ thống.
