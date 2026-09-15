# Báo cáo Ngày 3 — Tracking Annotation

Họ tên / nhóm: `Phạm Đại Phúc (Làm cá nhân)`
Ngày: `15/09/2026`

---

## 1. Quá trình gán nhãn

| Mục | Giá trị |
| --- | --- |
| Công cụ | CVAT |
| Thời gian gán `clip_02` (warm-up) | 25 phút |
| Thời gian gán `clip_01` | 75 phút |
| Số track đã vẽ trong `clip_01` | 8 |
| Số keyframe trung bình mỗi track | 5 - 10 |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. **Xe rời khỏi khung hình và lỗi bbox treo (Hanging Box):** Kết quả đối chiếu với Gold cho thấy bbox thường bị trôi thêm vài frame sau khi xe đã hoàn toàn khuất góc nhìn (điển hình ở ID 4, ID 8), làm tăng False Positive (FP = 60). Cách xử lý: Tua chính xác đến đúng frame xe biến mất và nhấn phím tắt `O` (`Switch outside`) để đóng track dứt điểm, ngăn box tồn tại ngoài phạm vi thực tế.
2. **Xe bị che khuất (Occlusion) bởi cây cối hoặc biển báo:** Khi xe đi qua các vật cản che khuất một phần thân, nguy cơ bị ngắt hoặc tách nhầm track rất cao (minh chứng qua việc ReID đạt AssA 0.820 tốt hơn ByteTrack chỉ đạt 0.776). Cách xử lý: Giữ nguyên `track_id`, bật thuộc tính `occluded = true` và tận dụng cơ chế nội suy tuyến tính (interpolation) của CVAT để duy trì danh tính thay vì tạo một track mới gây nhảy ID.
3. **Duy trì quỹ đạo khi xe thay đổi tốc độ hoặc dừng/đi chậm (Box Drifting):** Sự biến thiên vận tốc khiến thuật toán nội suy tự động bị rung lắc, mép hộp không ôm khít xe ở các frame ở giữa (LocA đạt 0.851/0.855). Cách xử lý: Bổ sung thêm các keyframe mấu chốt tại thời điểm xe bắt đầu giảm tốc, dừng hoặc đổi hướng di chuyển để nắn lại toàn bộ quỹ đạo của bounding box cho khít sát thân xe.

## 2. Tự kiểm và kiểm chéo

### Ba lượt tua bắt được gì:

* **Lượt 1 (Nhìn ID):** Quá trình theo dõi duy trì ID nhất quán rất tốt (ID Switch = 0). Tuy nhiên, phát hiện sự lệch pha về thời điểm bắt đầu khởi tạo track: ID 6 và ID 5 bị gán nhãn xuất hiện quá sớm so với thực tế (lần lượt sớm hơn 21 frame và 6 frame so với ground truth).
* **Lượt 2 (Frame đầu/cuối):** Phát hiện lỗi bbox treo (hanging box) ở cuối quỹ đạo chuyển động. Cụ thể, ID 4 và ID 8 vẫn còn tồn tại thêm 3 frame sau khi xe đã hoàn toàn đi ra khỏi khung hình. Cần thao tác bấm phím `O` (`Switch outside`) chuẩn xác ngay tại frame xe vừa biến mất.
* **Lượt 3 (Frame giữa):** Xuất hiện hiện tượng trôi hộp bao (drifting box) làm giảm độ trùng khớp IoU xuống ~0.51 ở các frame ở giữa (điển hình tại frame 111 của ID 6 và frame 93 của ID 5). Nguyên nhân do thuật toán nội suy tuyến tính giữa 2 keyframe chưa bám kịp khi xe thay đổi gia tốc/vận tốc.

### Ca nào hai bên quyết định khác nhau (Bạn vs Model ReID):

* **Điểm bất đồng tiêu biểu:** Tại frame 105 và 106, model ReID phát hiện được 2 đối tượng phương tiện mà người gán bỏ sót (dẫn đến chênh lệch chỉ số FN = 59 ở phép so sánh `reid_vs_ban`).
* **Lý do từ góc nhìn người gán:** Tại khu vực này, đối tượng ở quá xa mép ảnh và bị lóa sáng mạnh; người gán quan sát nhầm cụm chi tiết đó là ánh đèn phản chiếu/đèn đường thay vì một chiếc xe hoàn chỉnh nên đã quyết định bỏ qua không tạo track. Ngược lại, ReID trích xuất được vector đặc trưng diện mạo (appearance feature embedding) tương đồng với phương tiện nên vẫn duy trì nhận diện.

### Đề xuất bổ sung luật vào GUIDELINE_MINI.md:

* **Bổ sung ngưỡng nhận diện tại vùng biên (Border Truncation & Visibility Threshold):** Quy định rõ ràng *"Chỉ bắt đầu gán nhãn phương tiện khi nhìn thấy tối thiểu 20% diện tích thân xe hoặc bánh xe đã chạm rõ ràng vào mặt đường trong khung hình"*. 
* **Quy định về trường hợp lóa sáng/vật thể nhỏ ở xa:** Cần có tiêu chí phân biệt giữa nguồn sáng độc lập (cột đèn, bóng phản chiếu) và đèn pha xe cơ giới để tránh hiện tượng người gán bỏ sót (tăng FN) trong khi các mô hình trích xuất đặc trưng tự động vẫn bắt nhãn.

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence | Giá trị |
| --- | --- |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` | `8f65b8745b1332262f3f57e33c57fb8dc516cbc0df01d8ce95854fde5d532fff` |
| Thời điểm khóa | Hoàn thành trước khi mở reference Gold |
| Số row / frame / track trước khi mở reference | 618 row / 190 frame / 8 track |

| | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Bản pre-gold | 0.607 | 0.496 | 0.741 | 0.851 | 0.735 | 0.537 | 0.836 | 60 | 205 | 0 |
| Sau rework | 0.761 | 0.745 | 0.777 | 0.855 | 0.937 | 0.869 | 0.840 | 60 | 15 | 0 |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **CÓ**

Sau khi đọc danh sách lỗi, bạn đã sửa cụ thể những gì? Ghi theo frame và ID:

| Loại lỗi | Frame | ID | Đã sửa thế nào |
| --- | --- | --- | --- |
| Bbox treo | 149-151 | 4 | Cắt ngắn track, đặt trạng thái 'Outside' ngay khi xe rời khỏi khung hình. |
| Bbox thừa | 80-100 | 6 | Xóa các bbox dư thừa được vẽ trước khi xe thực sự xuất hiện. |
| Bbox trôi | 111 | 6 | Thêm keyframe tại frame 111 để chỉnh lại tọa độ (khắc phục IoU đang thấp ~0.51). |
| Bbox trôi | 93 | 5 | Thêm keyframe để bám sát thân xe hơn do nội suy không chính xác. |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục | Giá trị |
| --- | --- |
| Python / ultralytics / torch / lap | `3.13.15` / `8.4.145` / `2.11.0+cpu` / `0.5.13` |
| weights / hai tracker | `yolo26n.pt` / `bytetrack.yaml` & `botsort-reid.yaml` |
| conf / IoU / imgsz / classes | `0.25` / `0.70` / `960` / `[2, 5, 7]` |
| device | `cpu` |

| So sánh | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| bạn vs gold | 0.761 | 0.745 | 0.777 | 0.855 | 0.937 | 0.869 | 0.840 | 60 | 15 | 0 |
| ByteTrack control vs gold | 0.709 | 0.649 | 0.776 | 0.846 | 0.875 | 0.749 | 0.823 | 88 | 54 | 2 |
| BoT-SORT + ReID vs gold | 0.763 | 0.711 | 0.820 | 0.872 | 0.900 | 0.792 | 0.860 | 91 | 26 | 2 |
| ReID vs bạn | 0.713 | 0.659 | 0.774 | 0.852 | 0.882 | 0.765 | 0.834 | 81 | 61 | 3 |
## 5. Phân tích — năm câu hỏi

1. **MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**  
   * **So sánh:** Điểm của tôi có **IDF1 (0.937) cao hơn MOTA (0.869)**. Kết quả này phản ánh việc tôi duy trì danh tính đối tượng hoàn hảo với `IDSW = 0`. MOTA bị kéo thấp hơn chủ yếu do bị phạt bởi 60 lỗi False Positive (FP) khi vẽ dư ở mép biên.
   * **Ý nghĩa nếu MOTA cao mà IDF1 thấp:** Điều đó cho thấy detector/bộ gán nhãn bắt rất chuẩn vị trí của vật thể theo từng khung hình đơn lẻ (ít FP, ít FN), nhưng liên tục bị nhảy ID hoặc hoán đổi ID giữa các đối tượng (ID Switch cao, phân mảnh track).
   * **Vì sao MOTA không phạt nặng lỗi ID:** MOTA được định nghĩa theo công thức:
     $$\text{MOTA} = 1 - \frac{\sum (\text{FN} + \text{FP} + \text{IDSW})}{\sum \text{GT}}$$
     Trong đó, một lỗi `IDSW` chỉ bị tính là 1 điểm phạt duy nhất tại đúng frame diễn ra sự chuyển đổi ID, các frame sau đó nếu bắt đúng box thì vẫn tính là True Positive. Ngược lại, IDF1 đo lường sự nhất quán trên toàn bộ vòng đời của track; một lần đổi ID sẽ cắt đôi track và phạt nặng toàn bộ chuỗi định danh, khiến IDF1 nhạy cảm hơn nhiều với lỗi gán ID.

2. **ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**  
   * **So sánh chỉ số:**  
     * **IDF1:** BoT-SORT + ReID (0.900) vượt trội hơn ByteTrack (0.875).  
     * **AssA:** ReID (0.820) cao hơn đáng kể so với ByteTrack (0.776).  
     * **IDSW:** Cả hai mô hình đều có 2 lỗi ID Switch, tuy nhiên ReID giảm mạnh hiện tượng đứt gãy/tách track, thể hiện qua việc FN giảm sâu từ 54 xuống còn 26.
   * **Minh chứng frame sequence:** Tại chuỗi frame xung quanh **frame 113**, xe thuộc track 6 di chuyển qua khu vực bị che khuất và nhiễu nền. ByteTrack (chỉ dựa vào chuyển động hình học và IoU matching) bị mất dấu, dẫn đến việc ngắt track. Trong khi đó, BoT-SORT + ReID nhờ trích xuất vector đặc trưng diện mạo (appearance embedding) nên đã nhận diện lại đúng xe khi xuất hiện trở lại, duy trì liên kết track liền mạch.
   * **Lưu ý về Causal Effect:** Cần lưu ý rằng sự chênh lệch này **không cô lập hoàn toàn hiệu ứng nhân quả (causal effect) của riêng module ReID**. Nguyên nhân là do hai tracker có kiến trúc triển khai khác biệt toàn diện: ByteTrack sử dụng bộ lọc Kalman Filter tiêu chuẩn kết hợp đối sánh 2 tầng dựa trên IoU, trong khi BoT-SORT tích hợp thêm module bù trừ chuyển động của camera (Camera Motion Compensation - CMC) và biến đổi không gian bounding box ngoài việc thêm ReID.

3. **DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**  
   * **Sự thay đổi:** Chỉ số **DetA tăng từ 0.649 lên 0.711** khi chuyển sang ReID. **FN giảm hơn một nửa (từ 54 xuống 26)**, trong khi **FP tăng nhẹ (từ 88 lên 91)**. Việc FN giảm mạnh chứng minh khâu liên kết (Association) mạnh mẽ hơn đã giúp bộ theo dõi tự tin duy trì và "cứu" lại các bounding box có độ tin cậy thấp hoặc bị mờ.
   * **Phân tích lỗi còn lại:** Lỗi hiện tại phân bổ ở cả hai khâu nhưng **phần lớn nghiêng về Detector**:
     * **Lỗi Detector:** Vẫn tạo ra tới 91 FP do bộ phát hiện bắt nhầm các vật thể tĩnh ven đường hoặc ánh sáng phản xạ, đồng thời bỏ sót 26 FN ở các góc khuất sâu.
     * **Lỗi Association:** Vẫn còn tồn tại cục bộ với 2 lỗi ID Switch khi các xe đi cắt ngang nhau quá gần.

4. **Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**  
   * **Vị trí:** Frame 105–106 (ID 7 và ID 27 của model).
   * **Nguyên nhân:** Model ReID phát hiện và duy trì 2 đối tượng tĩnh này kéo dài tới gần 40 frame. Thực tế trên ảnh, đây chỉ là phần cột đèn/biển báo ven đường bị ánh sáng rọi vào tạo ảo ảnh thị giác (false detection từ detector). Tôi đã quan sát kỹ chuỗi chuyển động và quyết định chính xác khi không gán nhãn cho các vật thể tĩnh này, trong khi model bị bẫy và sinh ra FP.

5. **Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**  
   * **Vị trí:** Frame 149–151, ID 4.
   * **Nguyên nhân:** Model ReID chủ động kết thúc track của xe ID 4 ngay tại frame 148, trong khi nhãn ban đầu của tôi vẫn kéo dài box tới tận frame 151. Khi xem xét lại ảnh phóng to, xe đã hoàn toàn khuất khỏi mép khung hình từ frame 149; việc tôi để box trôi thêm 3 frame là lỗi bbox treo (hanging box). Evidence từ model ReID đã chỉ ra chính xác ranh giới kết thúc thực tế của đối tượng.

---

## 6. Nếu phải gán thêm 10 clip nữa

* **Sửa đổi trong `GUIDELINE_MINI.md`:**  
  * **Quy tắc biên rõ ràng:** Bổ sung điều khoản: *"Không khởi tạo hoặc duy trì bounding box cho xe khi diện tích hiển thị nhỏ hơn 10%, hoặc khi chỉ nhìn thấy một phần gương chiếu hậu/vệt sáng đèn pha sát mép ảnh"* nhằm loại bỏ triệt để lỗi bbox treo và giảm thiểu FP ở vùng biên.  
  * **Quy chuẩn gán nhãn vật thể tĩnh:** Định nghĩa rõ ngưỡng phân biệt giữa phương tiện giao thông đang dừng đỗ thực tế và các kết cấu hạ tầng/vật thể tĩnh dễ gây nhầm lẫn khi bị phản quang.

* **Thay đổi quy trình làm việc:**  
  * **Áp dụng chiến lược Model-Assisted / Peer-Review:** Cho chạy trước pipeline BoT-SORT + ReID trên video thô để đóng vai trò làm lớp rà soát sơ bộ (pre-annotation/peer-review).
  * **Kiểm tra tập trung vào điểm nóng:** Thay vì kiểm tra tuần tự từng frame, sẽ ưu tiên soi kỹ các frame mà model kích hoạt cờ cảnh báo tách track (`Track Split`) hoặc nhảy ID (`ID Switch`), vì đây chính là các phân đoạn có độ che khuất cao hoặc mật độ phương tiện đan xen phức tạp nhất.
## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt`
- [x] `annotations/clip_02/gt.txt`
- [x] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [x] `GUIDELINE_MINI.md` đã điền
- [x] `outputs/eval_vs_gold.json`
- [x] `outputs/model_bytetrack_clip_01.txt`
- [x] `outputs/model_reid_clip_01.txt`
- [x] `outputs/model_run_config.json`
- [x] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [ ] `reports/review_partner.md`
- [x] `reports/REPORT.md` (file này)