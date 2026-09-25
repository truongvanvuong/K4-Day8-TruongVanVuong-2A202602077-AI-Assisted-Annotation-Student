# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Trương Văn Vượng

Công cụ gán nhãn đã dùng: CVAT

Sao chép file này thành `reports/REPORT.md` rồi điền vào các chỗ trống. Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo hướng nào, và vì sao?

Dữ liệu hình ảnh trong bài toán này được trích xuất liên tục từ video (camera hành trình). Các khung hình (frames) nằm sát nhau về mặt thời gian sẽ có bối cảnh, góc máy và các phương tiện giao thông gần như y hệt nhau.

Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo hướng cao hơn thực tế. Nguyên nhân là do hiện tượng rò rỉ dữ liệu (data leakage): một khung hình trong tập test có thể chỉ cách một khung hình trong tập pool vài phần mười giây. Khi đó, mô hình chỉ cần "ghi nhớ" hình dáng, vị trí chính xác của chiếc xe ở bối cảnh đó (overfitting) là có điểm cao, chứ chưa thực sự học được cách nhận diện xe ở những hoàn cảnh mới.

Việc chia theo trục thời gian và để ra một khoảng vùng đệm (buffer) ở giữa đảm bảo tập test chứa những đoạn đường, chiếc xe và bối cảnh hoàn toàn độc lập, chưa từng xuất hiện trong tập pool. Khoảng đệm giúp cắt đứt sự tương đồng rớt lại giữa cuối đoạn pool và đầu đoạn test. Nhờ đó, số đo thu được mới phản ánh đúng khả năng tổng quát hóa (khả năng áp dụng thực tế) của mô hình.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Ở vòng 0 (khởi đầu lạnh), mô hình đạt điểm AP50 là 0.771. Nhìn vào độ phủ (recall) theo kích thước, xe nhỏ chỉ đạt 0.182, trong khi xe vừa đạt 0.547 và xe lớn đạt 0.561. Điều này chứng tỏ mô hình bỏ sót rất nhiều xe ở xa (kích thước nhỏ) nhưng làm khá tốt với các xe ở gần. Khi quan sát ảnh compare_round0.jpg, có một số vị trí khung AI vẽ bị lệch khỏi thân xe thực tế. Tuy nhiên, cần lưu ý rằng nhãn dùng để chấm điểm cũng do một mô hình tự động tạo ra, chưa hề có người kiểm duyệt qua từng khung hình; do đó, trong một số ca không khớp, có thể chính nhãn dùng để chấm bị sai chứ chưa chắc AI khởi đầu lạnh của bạn sai.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Chiến lược chọn ảnh dựa trên một công thức tính điểm tổng hợp cho mỗi ảnh. Một nửa số điểm (trọng số 0.5) đến từ việc AI không chắc chắn về vật thể. Ba phần mười (0.3) dựa trên việc AI vẽ nhiều khung nhưng còn rất lưỡng lự. Hai phần mười (0.2) cuối cùng là điểm khoảng cách thời gian so với các ảnh khác, với quy định hai ảnh trong cùng một lô ưu tiên phải cách nhau ít nhất 2 giây (MIN_GAP_S). Do góc quay camera liên tục, hai bức ảnh sát nhau sẽ gần như giống hệt nhau, không có ích gì khi học cả hai. Trong SELECTION.md, tôi đã chỉ ra 3 ảnh frame_0380.jpg, frame_0326.jpg và frame_0331.jpg có độ bất định rất cao để ưu tiên, đồng thời loại bỏ frame_0368.jpg (dù điểm khá cao) vì nó xuất hiện quá sát với một bức ảnh khác. Cuối cùng, cần nhớ rằng điểm cao chỉ mang ý nghĩa AI đang phân vân, nó không chứng minh rằng sau khi sửa xong bức ảnh đó thì AI sẽ nhận diện tốt hơn.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

Ở vòng 1, tôi đã gán nhãn 12 bức ảnh với tổng cộng 353 box: giữ nguyên 146 box, chỉnh sửa 10 box, xóa 13 box sai và thêm mới tới 197 box bị sót. Tuy nhiên, sau khi fine-tune, điểm AP50 vòng 1 tụt dốc thảm hại chỉ còn 0.581 (giảm 0.190 so với mức 0.771 ban đầu). Tất cả các nhóm xe đều suy giảm, đặc biệt recall của xe nhỏ giảm về 0. Khi nhìn compare_round1.jpg, ta thấy kết quả xấu đi rõ rệt, AI lúng túng và gán sót rất nhiều xe. Điều này phản ánh rõ sự khác biệt giữa 3 khía cạnh: những gì mắt tôi nhìn thấy bằng quan sát độc lập trong BLIND_SCAN.md, những thao tác sửa lỗi nhãn thực tế tôi làm trong REVIEW_LOG.csv, và việc AI có thực sự học được điều đó sau khi train hay không (rõ ràng mô hình đã học sai hướng). Một ca khó theo guideline là những chiếc xe ở xa chỉ còn 2 chấm đèn hoặc bị xe khác che khuất phần lớn.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Điểm AP50 của vòng 1 (0.581) đã giảm nghiêm trọng so với vòng khởi đầu lạnh (0.771). Với kết quả này, tôi quyết định dừng lại để đánh giá lại bộ nhãn thay vì tiếp tục làm vòng 2. Mô hình vẫn còn yếu ở việc nhận diện các xe ở quá xa và các xe bị lấp/cắt mép ở rìa ảnh. Việc rà nhãn cho các ca này rất tốn công và phải tránh chọn các ảnh sát nhau để không trùng lặp bối cảnh. Cũng cần nhắc lại các giới hạn của bài toán: tập kiểm thử quá nhỏ (chỉ 20 ảnh), có quy tắc bỏ qua xe quá nhỏ, và bản thân nhãn dùng để test vẫn chưa được rà soát thủ công 100%. Khi điểm bị giảm mạnh thế này, điều đầu tiên tôi cần kiểm tra là xem lại các khung nhãn mình vừa tự gán ở vòng 1 xem có sai lệch nghiêm trọng nào không trước khi ép AI train thêm.
