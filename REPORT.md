# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Trần Long Khánh` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602538` |
| GitHub | `khanhtrankuri` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `3871 / 6736` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `3` (6736, 6737, 6738) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,259 | 20,3 | BLOCK |
| IDM | 1,530 | 0,17 | baseline |
| EgoMLP | 0,763–0,765 | 16,4–30,8 | BLOCK |
| PercMLP | 0,624–0,630 | 0,8–1,6 | BLOCK |
| PercMLP+ (sau khi sửa) | 0,805 ± 0,011 | 0,28–0,31 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Dữ liệu có 992.679 dòng và 9.993 quỹ đạo thô; 383 quỹ đạo trùng hệt bị loại, còn 9.610 quỹ đạo sạch.
`Acc_LV` gần như không khớp động học: tương quan với `d(Speed_LV)/dt` chỉ 0,0454, trong khi cột xe theo sau đạt 0,9291.
Vì vậy tôi không dùng `Acc_LV`, mà lấy đạo hàm của `Speed_LV` rồi làm mượt để khai thác các ca xe dẫn phanh.
Chia theo khung hình sẽ để các khung gần nhau của cùng một quỹ đạo lọt vào cả train và test; chia theo quỹ đạo và loại trùng trước giúp ngăn rò rỉ, tránh một bài test dễ giả tạo.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Hai lần chạy cùng mã giống từng bit ở cả 600 đoạn. Bản float32/refactor khác bit ở 600/600 đoạn nhưng 0 đoạn vượt `atol=0,001`, `rtol=0,001`; sai khác cực đại chỉ 0,000338 m/s², nên đây là nhiễu số chứ không phải đổi hành vi.
AEB làm lệch 10/600 đoạn. Ở đoạn 61, khác biệt chỉ xuất hiện tại khung 98–101: TTC lần lượt xấp xỉ 1,47; 1,43; 1,43; 1,49 s, vừa dưới ngưỡng 1,5 s, nên lệnh được chặn xuống −6 m/s².
Chạy nhanh nhất có 0 trễ và 0 rớt vì không áp deadline. Phát theo thời gian thật làm lộ 585 khung trễ và 583 khung rớt trên 60.018 khung, khiến 384/600 đoạn vượt dung sai: giữ lệnh cũ khi quá hạn là một thay đổi hành vi thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới dùng tốc độ cruise 14 m/s, khoảng cách 2,76–59,31 m và tốc độ tiến gần 0–14 m/s, tổng cộng 60×45 = 2.700 ô.
Không trễ, 3,63% lưới là bất khả thi; với trễ 0,6 s con số tăng lên 10,22%, tức tăng 6,59 điểm %. IDM đâm đúng hai vùng này và có 0% va chạm quy trách nhiệm, nên các cú đâm đó không phải thất bại riêng của bộ điều khiển.
Từ log chỉ khai thác được 614 ca phanh (6,39% quỹ đạo); giới hạn phanh người theo sau ở phân vị 99,9 là 3,62 m/s², thấp xa mức cơ cấu chấp hành 9 m/s².
Có 12,67% ô đòi phanh mạnh hơn mọi ví dụ người lái đã cho mô hình thấy; vì thế các chấm log chủ yếu phủ vùng quen thuộc và không đủ đại diện cho biên cut-in khó.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 (m), hạng | Va chạm vòng kín, hạng |
|---|---:|---:|
| constant-velocity | 1,259; #3 | 20,33%; #3 |
| IDM | 1,530; #4 | 0,17%; #1 |
| EgoMLP | 0,764; #2 | 24,03% trung bình; #4 |
| PercMLP | 0,627; #1 | 1,18% trung bình; #2 |
Các cặp đảo thứ tự là constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP. IDM dự báo quỹ đạo người thật không sát nên L2 kém, nhưng phản hồi trực tiếp theo gap và tốc độ tương đối nên ổn định khi tự lái. EgoMLP bắt chước trung bình tốt hơn constant-velocity nhưng không nhìn thấy xe dẫn/gap; sai số tích lũy làm nó không biết lúc nào cần phanh.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động từ 16,41% (473/2.883) đến 30,84% (889/2.883), biên độ 14,43 điểm % chỉ do đổi seed huấn luyện.
Seed tốt nhất có CI95 15,10–17,80%, còn seed xấu nhất 29,18–32,55%; hai khoảng hoàn toàn không chồng nhau, nên đây không chỉ là nhiễu lấy mẫu nhỏ.
Nếu chỉ chạy seed 6736 tôi có thể báo “16,4%”, nhưng nếu tình cờ chạy 6738 lại phải báo “30,8%”, dẫn đến hai đánh giá rất khác nhau về cùng kiến trúc.
Vì vậy phải báo dải qua nhiều seed và dùng trường hợp xấu nhất ở cổng; một seed đẹp có thể che độ bất ổn và đánh giá thấp rủi ro triển khai.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Người thật có headway trung vị 1,88 s; hiệu chỉnh chọn `T=1,4 s` vì khoảng cách Wasserstein thấp nhất 0,362, tốt hơn nhiều xe không phản ứng (0,839).
Ở trễ 0 s, tỷ lệ “bất kỳ va chạm” replay/react lần lượt là: constant-velocity 80,6/80,6%; IDM 80,3/3,6%; EgoMLP 86,7/86,7%; PercMLP 75,2/19,3%.
Ở trễ 0,6 s, các cặp tương ứng là 80,6/80,6%; 80,9/10,2%; 85,5/85,5%; 78,2/33,3%.
Xe sau replay cứ giữ tốc độ dù xe thử đã phanh nên “đâm sau” IDM ở 76,7% (trễ 0) và 70,6% (trễ 0,6); khi cho nó phản ứng, cả hai về 0%. Vì vậy thế giới replay làm IDM trông gần tệ như constant-velocity, còn thế giới phản ứng cho thấy IDM an toàn hơn hẳn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị BLOCK ở R2 (1,6%, cận trên 2,2% so với IDM 0,2% + 1 điểm), R3 (5,8%, cận trên 10,4%), R4 (23,1% vùng cut-in tránh được vẫn đâm) và R6 (33,3% so với IDM 10,2%). R4/R6 cho thấy thiếu kịch bản cut-in ngoài log; R2/R3 cho thấy sai số bắt chước tích lũy khi tự lái.
Tôi thêm 197.150 cửa sổ từ 20.000 cut-in mô phỏng có nhãn IDM, rồi chạy 2 vòng DAgger để gắn nhãn cho chính các trạng thái lệch mà model đi vào. Sau sửa, PercMLP+ đạt 0,3% va chạm log, 0,9% vùng cut-in quy trách nhiệm và qua cả 7 quy tắc.
Tuy nhiên L2 xấu đi từ 0,627 lên 0,805 m, minh họa rằng điểm vòng hở không phản ánh trực tiếp độ an toàn vòng kín. PercMLP+ học từ IDM nên bằng chứng chỉ chứng minh “không kém hơn” theo cổng, không chứng minh tốt hơn: IDM vẫn là 0,17% va chạm và 0% vùng quy trách nhiệm.
Tôi chưa thay IDM ngay. Cần kiểm thử độc lập ở phân phối mới—tại cruise 20 m/s, PercMLP+ vẫn có seed đâm 4,11% vùng tránh được trong khi IDM là 0%—sau đó shadow test, tăng độ phủ và đánh giá trên hệ thống thật trước quyết định phát hành.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Thứ nhất, mô phỏng chỉ có chuyển động dọc 1D nên không kiểm tra chuyển làn, góc cắt vào, cong đường hay tương tác nhiều xe; 2.700 ô không đại diện cho không gian giao thông thật. Cần mô phỏng đa tác nhân và scenario replay 2D/3D đã hiệu chỉnh.
Thứ hai, bài bỏ qua cảm biến và perception thực: gap/tốc độ xe dẫn gần như trạng thái đã biết, còn thử nhiễu chỉ dùng độ lệch 1 m và 0,5 m/s. Cần log cảm biến thô, lỗi bỏ sót/nhầm vật thể, thời tiết, che khuất và hardware-in-the-loop với timing thật.
Thứ ba, xe dẫn chủ yếu được replay và mô hình xe sau phản ứng chỉ là IDM `T=1,4 s`; con người thật có chiến lược đa dạng và phản ứng hai chiều. Cần naturalistic/closed-course test với nhiều tác nhân phản ứng đã kiểm định.
Cuối cùng, dữ liệu phi thương mại chỉ có 9.610 quỹ đạo sạch và log phanh phủ 614 ca; ngay ở tốc độ ngoài train 20 m/s, PercMLP+ còn dải lỗi 0,04–4,11%. Vì vậy RELEASE của cổng lab chỉ là đạt tiêu chí mô phỏng, không phải chứng nhận an toàn; cần mở rộng ODD, stress test, shadow deployment và giám sát sự cố trước phát hành.
