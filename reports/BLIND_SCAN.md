# Quét độc lập trước khi xem pre-label

Frame:`frame_0312.jpg`

Số xe nhìn thấy bằng mắt: 22 xe

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:

- Vị trí 1: Chiếc xe tối màu đang đi tới ở làn giữa, gần phần dưới khung hình. Xe này bật đèn pha trắng cực kỳ chói lóa, tạo vệt phản chiếu dài trên mặt đường. AI dễ vẽ sai bounding box/polygon (thường bị vẽ quá to, bao trùm cả quầng sáng lóa flare và vệt sáng dưới đường thay vì chỉ bám sát viền thân xe thực tế), hoặc bóp méo hình dáng đầu xe do bị cháy sáng.
- Vị trí 2: Chiếc xe ở tít hậu cảnh phía xa góc trái (gần dải phân cách). Đây chỉ là một cái bóng đen rất mờ chìm vào nền đêm, nhận diện chủ yếu bằng hai chấm đỏ cực mờ của đèn hậu. Do kích thước pixel quá nhỏ (small object) và độ tương phản với nền cực kỳ thấp, AI (pre-label) rất dễ bỏ sót hoàn toàn chiếc xe này (False Negative).

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
