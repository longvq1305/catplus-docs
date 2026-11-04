# Báo cáo kỹ thuật: Phân tích nguyên nhân và quy trình xử lý lỗi đồng bộ bài tập (29/10/2026)

## 1. Vấn đề phát sinh và nguyên nhân gốc rễ
Hệ thống CatPlus Online Judge kết nối với nền tảng Polygon để tiếp nhận, quản lý và cập nhật nội dung bài tập cùng dữ liệu chấm thi. Khi người quản trị chỉnh sửa bài tập, lưu lại phiên bản đã chốt (thao tác commit) và tạo tệp nén đầy đủ cho môi trường Linux (gói full package) ở trạng thái sẵn sàng (`READY`), thao tác đồng bộ lại vẫn có thể bị dừng ở trạng thái chờ chốt phiên bản (`WAITING_FOR_COMMIT`), khiến dữ liệu bài tập không thể cập nhật vào hệ thống.

Trong quá trình đồng bộ, trạng thái bài tập liên quan đến ba yếu tố thông tin:

- **Phiên bản đã chốt:** Phiên bản chính thức đã được lưu trong lịch sử bài tập trên Polygon.
- **Phiên bản của gói bài:** Phiên bản của tệp nén dữ liệu hoàn chỉnh (gói full package cho Linux) ở trạng thái sẵn sàng (`READY`), chứa toàn bộ nội dung đề bài, hình ảnh và bộ dữ liệu kiểm thử.
- **Phiên bản nội dung đề trả về:** Phiên bản của nội dung đề do Polygon trả về riêng biệt khi hệ thống gửi yêu cầu đọc trực tiếp.

Nguyên nhân gây tắc nghẽn bắt nguồn từ điều kiện kiểm tra trước đây: dù hệ thống đã bỏ qua cảnh báo bản nháp khi có gói bài hoàn chỉnh, một bước kiểm tra tiếp theo lại quy định nếu phiên bản của nội dung đề trả về bị lệch so với phiên bản đã chốt thì hệ thống tự động coi như bài tập chưa được chốt và chuyển về trạng thái chờ (`WAITING_FOR_COMMIT`).

Trong thực tế vận hành, việc nội dung đề trả về riêng lẻ bị lệch phiên bản không đồng nghĩa với việc bài tập chưa được chốt. Ví dụ trong kịch bản kiểm thử có kiểm soát, bài tập đã ở phiên bản 7 và gói bài phiên bản 7 đã sẵn sàng (`READY`), nhưng phản hồi đọc đề riêng lẻ lại được giả lập trả về phiên bản 6 (đây là tình huống kiểm thử mô phỏng có kiểm soát, không phải nhật ký ghi nhận nguyên nhân từ hệ thống Polygon thực tế). Việc quy kết sự sai lệch này thành tình trạng chưa chốt phiên bản đã khiến tiến trình đồng bộ dừng lại không cần thiết dù gói bài hợp lệ của phiên bản mới đã sẵn sàng.

Để giải quyết sự cố trên, nội dung báo cáo tập trung vào giải pháp xử lý trực tiếp lỗi dừng đồng bộ ở trạng thái chờ chốt phiên bản (Mục 2), đồng thời làm rõ hai quy trình vận hành và bảo trì phụ trợ liên quan: cơ chế chạy ngầm khi đồng bộ theo danh mục (Mục 3) và quy trình tùy chọn để tái tạo nội dung hiển thị từ dữ liệu đã lưu (Mục 4). Hai quy trình phụ trợ này hỗ trợ sự ổn định trong vận hành thực tế, không phải là nguyên nhân gây lỗi hay các bước bắt buộc để giải phóng trạng thái chờ chốt phiên bản.

## 2. Quy trình xử lý đồng bộ bài tập đã chỉnh sửa
Tiến trình đồng bộ một bài tập sau khi chỉnh sửa được thực hiện tuần tự qua các bước sau:

1. **Tiếp nhận yêu cầu đồng bộ:** Hệ thống ghi nhận bài tập cần đồng bộ theo yêu cầu từ người dùng.
2. **Tra cứu thông tin và danh mục gói bài:** Hệ thống gửi yêu cầu sang Polygon để đọc thông tin bài tập, lấy phiên bản đã chốt trên Polygon làm phiên bản mục tiêu và tra cứu danh mục gói bài để tìm đúng gói bài hoàn chỉnh cho Linux (gói full package) tương ứng với phiên bản này.
3. **Chọn cách lấy nội dung đề:** Hệ thống kiểm tra phản hồi nội dung đề từ Polygon. Nếu nội dung đề trả về đúng phiên bản đã chốt, hệ thống sử dụng trực tiếp dữ liệu này. Nếu nội dung đề bị lệch phiên bản, hệ thống chuyển sang quy trình dự phòng: sử dụng gói bài hoàn chỉnh của đúng phiên bản đã chốt đang ở trạng thái sẵn sàng (`READY`), tải từ Polygon về hoặc dùng lại gói đã lưu tại máy chủ nếu đã được xác thực hợp lệ.
4. **Kiểm tra gói bài khi dùng nguồn dự phòng:** Khi phải sử dụng gói bài dự phòng, hệ thống đọc nội dung đề dạng văn bản, tệp hình ảnh đính kèm, tiêu đề cùng các giới hạn thời gian và bộ nhớ từ tệp nén. Tệp cấu trúc mô tả bài tập bên trong gói được kiểm tra chặt chẽ: phiên bản công bố bên trong phải khớp chính xác với phiên bản đã chốt, các tệp bắt buộc phải có mặt đầy đủ, kích thước và đường dẫn tệp phải nằm trong giới hạn an toàn. Nếu không tìm thấy gói bài phù hợp, gói bài chưa đạt trạng thái sẵn sàng hoặc nội dung gói nén không vượt qua các bước kiểm tra an toàn, hệ thống lập tức dừng tiến trình và thông báo lỗi rõ ràng; tuyệt đối không ép buộc cập nhật bằng gói bài sai phiên bản hoặc gói cũ.
5. **Cập nhật dữ liệu và bảo toàn lịch sử:** Sau khi dữ liệu đề bài được xác thực an toàn (từ nội dung đề nhận trực tiếp hoặc trích xuất từ gói dự phòng), hệ thống làm sạch nội dung và ghi nhận dữ liệu mới để phục vụ hiển thị. Toàn bộ lịch sử các phiên bản trước đây và dữ liệu các lần chấm bài cũ được giữ nguyên vẹn.

## 3. Quy trình tổ chức công việc nền và xử lý gián đoạn
Quy trình ở Mục 2 áp dụng cho từng bài tập. Khi đồng bộ cả một danh mục, hệ thống cần thực hiện các bước đó cho nhiều bài, và việc đọc thông tin, kiểm tra, tải gói từ Polygon có thể kéo dài. Do đó, hệ thống cần một cơ chế công việc nền để máy chủ tiếp tục xử lý độc lập với giao diện người dùng, đồng thời ghi nhận tiến độ và hỗ trợ gửi lại hoặc tiếp tục công việc khi gặp sự cố gián đoạn mạng.

Nếu Mục 2 giải thích cách chọn và kiểm tra dữ liệu của từng bài, thì phần này giải thích cách tổ chức và theo dõi các bước đó cho cả danh mục. Trình tự tổ chức hàng đợi công việc và xử lý gián đoạn được thực hiện như sau:

- **Tiếp nhận và xếp hàng công việc:** Khi người quản trị yêu cầu đồng bộ danh mục bài tập, máy chủ ghi nhận yêu cầu vào hàng đợi với trạng thái chờ (`PENDING`) và phản hồi ngay cho giao diện người dùng. Tiến trình xử lý nền sau đó mới tiếp nhận công việc, quét danh mục bài tập và tạo các công việc con cho từng bài riêng lẻ.
- **Kiểm soát quyền xử lý và xử lý quá hạn:** Tiến trình nền nắm giữ quyền xử lý trong một khoảng thời gian giới hạn và định kỳ gửi tín hiệu xác nhận vẫn đang hoạt động bình thường. Nếu quá trình khởi tạo danh mục không có phản hồi và đã quá thời gian cho phép trước khi kịp tạo xong các công việc con cho từng bài, hệ thống đánh dấu đợt chạy là thất bại (`FAILED`) và hủy quyền xử lý để người quản trị có thể chủ động kích hoạt đợt chạy mới; hệ thống không tự động đưa lại vào hàng đợi.
- **Lưu thông tin để gửi lại khi cần:** Trước khi gửi dữ liệu cập nhật sang hệ thống quản lý bài tập, hệ thống lưu lại mã nhận biết và nội dung yêu cầu. Khi gặp lỗi kết nối mạng tạm thời, hệ thống sử dụng đúng mã nhận biết và nội dung đã lưu để gửi lại, đồng thời ghi nhận tiến độ bước đang làm để tiếp tục công việc mà không tạo ra các bản ghi trùng lặp.
- **Đồng bộ trạng thái trên giao diện:** Khi người dùng tải lại trang web, giao diện tự động kết nối lại vào tiến trình đang hoạt động để hiển thị tiến độ. Nếu kết nối mạng giữa trình duyệt và máy chủ bị gián đoạn, giao diện cung cấp nút cho người dùng chủ động bấm tiếp tục theo dõi khi mạng phục hồi, không tự ý gửi thêm yêu cầu chạy mới; việc đóng trình duyệt không làm gián đoạn hay hủy bỏ công việc nền đang chạy trên máy chủ.

## 4. Quy trình tái tạo đề bài và quản lý bản hiển thị
Gói bài lấy về trong quá trình đồng bộ không chỉ phục vụ việc cập nhật dữ liệu mà còn có thể được dùng lại khi cần tạo lại nội dung đề để hiển thị cho người đọc. Khi cần tạo lại cách hiển thị đề đã có — chẳng hạn để áp dụng định dạng hiển thị công thức toán mới, căn chỉnh lại bố cục trang hoặc lọc các thành phần theo tiêu chuẩn an toàn hiện tại — hệ thống có thể tận dụng ngay tệp gói nén đã lưu trữ sẵn trên máy chủ để tạo lại nội dung đề bài mà không cần gửi thêm yêu cầu tải lại từ Polygon.

Đây là thao tác bảo trì nội bộ tùy chọn, không phải là bước bắt buộc sau mỗi lần đồng bộ và không thể thay thế cho quy trình đồng bộ bài tập ở Mục 2. Quy trình này hoàn toàn không lấy bản cập nhật mới trên Polygon, không tự sửa lỗi dừng ở trạng thái chờ chốt phiên bản và không làm thay đổi dữ liệu chấm thi, mà chỉ phục vụ việc làm mới nội dung hiển thị và quản lý bản đề được phát hành chính thức theo trình tự sau:

- **Kích hoạt và xác thực quyền hạn:** Người quản trị kích hoạt chức năng tái tạo toàn bộ đề bài thông qua nút bấm trên giao diện kèm hộp thoại xác nhận. Quyền hạn của người dùng được kiểm tra chặt chẽ ở cả giao diện và phía máy chủ trước khi thực thi.
- **Đọc gói bài nội bộ và tái tạo đề từ gói đã lưu:** Hệ thống đọc lại tệp gói bài nén đang lưu trên máy chủ, đối chiếu thông tin để xác nhận tệp còn nguyên vẹn mà không gửi bất kỳ cuộc gọi mạng nào sang Polygon. Quá trình này hoàn toàn không làm thay đổi bộ dữ liệu chấm hay thông tin chấm thi của bài tập.
- **Định dạng công thức toán và làm sạch dữ liệu:** Hệ thống định dạng lại nội dung đề bài, hỗ trợ hiển thị các cấu trúc văn bản và biểu thức toán học lồng nhau, tự căn chỉnh phù hợp trên các màn hình hẹp, đồng thời lọc sạch các thành phần không an toàn trước khi lưu.
- **Chọn bản đề hiển thị và ghi nhận lịch sử:** Hệ thống lưu nội dung đề sau xử lý và ghi nhận bản được dùng để hiển thị cho từng phiên bản và ngôn ngữ tương ứng. Lịch sử các lần tạo đề trước được giữ cố định. Dựa vào thời điểm khởi tạo, hệ thống ngăn không cho một tác vụ cũ ghi đè lên bản đề mới hơn. Nếu một bài tập bị thiếu gói nén lưu trữ, hệ thống ghi nhận lỗi riêng cho bài đó và tiếp tục xử lý các bài còn lại trong danh mục.
- **Thống nhất trải nghiệm hiển thị:** Khi lựa chọn cùng phiên bản và cùng ngôn ngữ, giao diện quản trị và giao diện học viên cùng đọc nội dung từ cùng bản đề đã được chọn để phát hành.

## 5. Phương pháp kiểm chứng và kết quả thử nghiệm
Phương án xử lý đã trải qua các kịch bản kiểm thử có kiểm soát trên môi trường thử nghiệm cô lập:

- **Đồng bộ lại khi nội dung đề trả về bản cũ:** Trong kịch bản kiểm thử giả lập khi phiên bản nội dung đề trả về bị cũ hơn phiên bản đã chốt và gói bài hoàn chỉnh đã sẵn sàng (`READY`), hệ thống tự động kích hoạt quy trình dự phòng, trích xuất chính xác dữ liệu từ gói bài của phiên bản đã chốt và lưu giữ đầy đủ lịch sử các phiên bản.
- **Dừng an toàn khi gói bài không hợp lệ:** Khi tệp cấu trúc bài tập bị lỗi hoặc thiếu tệp đề bài trong gói nén, hệ thống từ chối cập nhật và đưa ra thông báo lỗi rõ ràng.
- **Xử lý nền và tái tạo ngoại tuyến:** Tác vụ đồng bộ danh mục được đưa vào hàng đợi và xử lý trọn vẹn ở tiến trình nền; chức năng tái tạo đề bài hoàn tất mà không phát sinh cuộc gọi mạng nào sang Polygon, đồng thời cơ chế ngăn tác vụ cũ ghi đè dữ liệu hoạt động chính xác.
- **Thử lại khi gặp sự cố mạng:** Mã nhận biết và nội dung yêu cầu được bảo toàn qua các lần gửi lại sau sự cố mất kết nối mạng mô phỏng.
- **Kiểm tra giao diện và trao đổi dữ liệu:** Giao diện hiển thị tốt trên cả máy tính lẫn thiết bị di động, kiểm tra kiểu dữ liệu tĩnh đạt chuẩn và dữ liệu trao đổi giữa giao diện và dịch vụ nền bảo đảm tính thống nhất.

Kết quả kiểm tra phần mềm ghi nhận: các kịch bản chính của dịch vụ đồng bộ bài tập cùng dịch vụ quản trị và chấm bài đều đã được kiểm chứng. Giao diện người dùng đã hoàn tất các bước biên dịch mã nguồn, kiểm tra kiểu dữ liệu tĩnh và kết xuất trang tĩnh. Có hai bài kiểm tra của dịch vụ quản trị và chấm bài cùng bước đóng gói giao diện chưa đạt do môi trường thử nghiệm Windows thiếu quyền tạo liên kết tệp đặc thù.

## 6. Giới hạn kỹ thuật và phạm vi vận hành thực tế
Báo cáo ghi nhận khách quan các giới hạn kỹ thuật và phạm vi vận hành thực tế của giải pháp:

- **Trình tự tương tác mạng:** Yêu cầu kiểm tra thông tin bài và đọc nội dung đề từ Polygon luôn diễn ra trước khi quy trình dự phòng có thể được kích hoạt. Nếu kết nối mạng sang Polygon bị gián đoạn ngay từ các bước gọi ban đầu, tiến trình đồng bộ vẫn sẽ dừng lại dù gói bài đã có sẵn trên máy chủ.
- **Phạm vi của quy trình dự phòng:** Cơ chế đọc đề trực tiếp từ gói nén chỉ hoạt động khi gói bài đầy đủ của đúng phiên bản đã chốt đã được tạo xong và đạt trạng thái sẵn sàng (`READY`). Quy trình này không thể thay thế cho trường hợp gói bài chưa được tạo hoặc bản thân gói bài bị hỏng.
- **Hiển thị công thức toán và an toàn:** Hệ thống hỗ trợ tốt hơn các dạng công thức toán học lồng nhau và lọc nội dung nguy hiểm, nhưng không tương đương với một bộ phân tích toàn diện toàn bộ cú pháp toán học hay cam kết loại bỏ tuyệt đối mọi nguy cơ an toàn.
- **Phạm vi môi trường kiểm chứng:** Toàn bộ quá trình kiểm chứng được tiến hành trên môi trường thử nghiệm nội bộ với dịch vụ Polygon giả lập và cơ sở dữ liệu thử nghiệm. Cần kiểm chứng thêm khi nhiều tác vụ cùng cập nhật trên PostgreSQL; đồng thời chưa thực hiện đồng bộ với Polygon thực tế trong đợt kiểm tra này.
