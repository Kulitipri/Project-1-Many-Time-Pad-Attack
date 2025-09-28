# Project1 - Many Time Pad Attack
Mục tiêu ngắn gọn

Bạn có một bộ ciphertexts được mã bằng stream cipher (hoặc OTP) mà keystream bị reuse cho nhiều plaintext. Mục tiêu là tái tạo plaintext (đặc biệt target) bằng cách khai thác mối quan hệ:

ct_i = pt_i ⊕ key
ct_j = pt_j ⊕ key
=> ct_i ⊕ ct_j = pt_i ⊕ pt_j


Từ ct_i ⊕ ct_j ta có thông tin về tương quan giữa pt_i và pt_j — đặc biệt là nếu một trong hai ký tự là space (0x20).

Ý tưởng thuật toán (tóm tắt quá trình tư duy)

XOR mọi cặp ciphertexts để thu pt_i ⊕ pt_j.

Tại mỗi vị trí pos, nếu ct_i ⊕ ct_j là ký tự chữ (A–Z hoặc a–z), đó là dấu hiệu rằng ít nhất một bên ở vị trí đó có space (vì space ⊕ letter → letter-like).

Đếm số lần một ciphertext ct_i khi XOR với các ciphertext khác tạo ra chữ ở pos. Nếu ct_i được “vote” đủ nhiều (>= threshold), ta giả sử pt_i[pos] = ' ' và suy ra key[pos] = ct_i[pos] ⊕ 0x20.

Bây giờ có partial_key ở một số vị trí; dùng partial_key để decrypt các ciphertexts (chỗ chưa biết in ?).

(Optional) Space trick (fill): với các vị trí chưa có key, giả sử từng ciphertext 'j' có space tại pos ⇒ ứng viên candidate = cts[j][pos] ⊕ 0x20. Áp candidate này lên các ciphertext khác, đếm xem bao nhiêu plaintext ký tự hợp lệ — chọn candidate có “ủng hộ” nhiều nhất. (điều này lấp nhiều ? nhưng có thể gây sai)

(Optional) Manual hints: nếu bạn có kiến thức chắc chắn (ví dụ biết phần đầu một ciphertext khác), tính toán đúng key byte bằng key[pos] = cts[i][pos] ⊕ ord(known_char) — đây là phép toán, không phải đoán mò.

Phần toán học (nhỏ, minh hoạ)

Với ct = pt ⊕ key, suy ra pt = ct ⊕ key.

Nếu pt_i[pos] = ' ' (0x20) thì key[pos] = ct_i[pos] ⊕ 0x20.

Ví dụ: nếu ct_i[pos] = 0x35 và bạn biết pt_i[pos] = 'a' (0x61), thì key[pos] = 0x35 ⊕ 0x61.

Quan trọng: ct_i[pos] ⊕ ct_j[pos] = pt_i[pos] ⊕ pt_j[pos] — ta chỉ sử dụng phép XOR này để phát hiện khả năng space/letter.

Phân tích từng phần code bạn đưa (giải thích từng khối)

Mình phân tích theo thứ tự code của bạn (phiên bản cuối cùng bạn gửi).

1. Khai báo dữ liệu
ct_hex = [ct1, ct2, ..., target_ct]
cts = [binascii.unhexlify(h) for h in ct_hex]


Chuyển hex → bytes để xử lý XOR. cts là danh sách các ciphertext (bytes).

cts[-1] là target (bản cần giải).

2. Hàm XOR
def xor_bytes(a: bytes, b: bytes) -> bytes:
    return bytes([a[i] ^ b[i] for i in range(min(len(a), len(b)))])


XOR chỉ chạy tới min(len(a), len(b)).

Hệ quả: phần đuôi của chuỗi dài hơn không được so sánh với chuỗi ngắn hơn.

Lý do thường dùng: tránh IndexError; nhưng nếu bạn muốn “so sánh toàn diện” cần xử lý padding hoặc logic khác.

3. Hàm kiểm tra ký tự “alphabet-like”
def is_alpha_byte(b: int) -> bool:
    return (65 <= b <= 90) or (97 <= b <= 122) or (b == 32)


Trả True cho A–Z, a–z và space.

Khi ct_i ⊕ ct_j cho ra giá trị nằm trong khoảng này → ta coi đó là dấu hiệu space/letter.

4. analyze_threshold(threshold)

Mục tiêu: tìm các vị trí có khả năng là space cho từng ciphertext.

Tạo ma trận counts[i][pos] = số lần ct_i ⊕ ct_j cho kết quả alphabet ở pos.

Nếu counts[i][pos] >= threshold → cho rằng ct_i có space ở pos.

Suy ra partial_key[pos] = cts[i][pos] ^ 0x20.

Ý nghĩa thực tế:

threshold là mức “độ tin cậy” — càng cao thì càng ít false positive, nhưng sẽ có nhiều ?.

Kết quả partial_key là 1 tập các vị trí key có độ tin cậy vừa đủ theo threshold.

5. decrypt_with_key(ct, key)
return ''.join(chr(ct[pos] ^ key[pos]) if pos in key else '?' for pos in range(len(ct)))


Dùng key để giải từng ciphertext; vị trí chưa có key trở thành '?'.

6. Manual hints

Bạn thêm:

manual_hints = [
  (0,7,'f'), (1,25,'y'), ...
]
for ci,pos,ch in manual_hints:
    key[pos] = cts[ci][pos] ^ ord(ch)


Với mỗi hint, bạn tính chính xác 1 byte của key dựa trên ciphertext được biết plaintext 1 ký tự.

Nếu ci = 10 (target), bạn đang “nhập” phần target làm nguồn dữ liệu (tức bạn đã biết câu trả lời); điều này hợp pháp khi bạn đang kiểm thử/điều chỉnh, nhưng nếu mục tiêu là tự động thì không nên dùng target.

7. Tiếp tục: in kết quả

Trước khi thêm human predictions bạn in target; sau khi thêm bạn in lại — mục đích so sánh.

Vì sao một số đoạn (ví dụ "using a stream") khó khôi phục tự động?

Mình tóm lại các nguyên nhân chính đã bàn trước, nhưng diễn giải thêm:

Thiếu bằng chứng “space” đồng thời trong nhiều ciphertexts

analyze_threshold cần nhiều cặp ct_i ⊕ ct_j để vote; nếu phần văn bản tương ứng chỉ xuất hiện trong 1 ciphertext (hoặc xuất hiện nhưng luôn là letter ở mọi ciphertext khác), thì không có bằng chứng cho space.

Xung đột do fill/heuristic

Nếu bạn dùng thêm fill_key_with_space_trick (phiên bản khác) thì nó sẽ ép chọn candidate cho mọi vị trí. Candidate được chọn dựa trên “số ciphertext ủng hộ” — nhưng nếu không có majority rõ ràng, candidate sai sẽ được chọn → ký tự rác.

Độ dài ciphertext khác nhau

Vị trí ở cuối ciphertext dài hơn có ít phép so sánh với các ciphertext ngắn hơn → ít evidence → dễ bị ? hoặc chọn sai.

Không dùng ngữ cảnh ngôn ngữ

Phương pháp chỉ dựa trên bitwise thống kê, không biết “using a stream” là cụm từ hợp lệ. Vì thế chỗ không có evidence không thể tự sửa.

Phần “manual hint” — nó là tính toán, không phải đoán bừa

Một manual hint cố định một byte plaintext cho một ciphertext ct_i. Từ đó key[pos] được tính bằng XOR chính xác: key[pos] = cts[i][pos] ^ ord(ch).

Vì vậy manual hints là ràng buộc toán học chắc chắn cho keystream ở vị trí đó.

Một manual hint đúng có thể “kéo” cả hệ thống về hướng chính xác (vì key là chung cho tất cả ciphertext).

Nhận xét về hiệu quả của threshold trong code hiện tại

threshold chỉ tạo seed ban đầu (partial_key).

Nếu bạn KHÔNG dùng fill thì threshold có tác động lớn đến kết quả (thấp → nhiều chữ lộ ra + nhiều sai; cao → ít chữ nhưng ít sai).

Nếu bạn có fill_key_with_space_trick nó sẽ lấp phần lớn vị trí chưa có key, làm cho thay đổi threshold ít ảnh hưởng tới kết quả cuối cùng (vì fill ghi đè / lấp các chỗ trống).

Những điểm yếu & rủi ro trong code hiện tại (tổng hợp)

xor_bytes dùng min(len(a), len(b)) → phần đuôi không được so sánh.

is_alpha_byte quá đơn giản — nên mở rộng kiểm tra printable ASCII khi cần.

fill_key_with_space_trick có thể ghi đè key tốt ban đầu — cần tránh overwrite key do threshold đã xác định.

Manual hints dùng target => làm mất tính “tự động”.

Không có kiểm tra xung đột khi hai hint gán key khác nhau cho cùng vị trí.

Không lưu key giữa các lần chạy (khó reproduce).

Gợi ý cải tiến (cụ thể, code snippets / ý tưởng)

Không overwrite key được seed từ threshold:

# khi fill, kiểm tra nếu pos đã có key thì skip
if pos in key:
    continue


Xử lý conflict (báo nếu một hint cố gắng ghi đè byte khác):

def apply_hint(key, ci, pos, ch):
    new_k = cts[ci][pos] ^ ord(ch)
    old = key.get(pos)
    if old is not None and old != new_k:
        print(f"Conflict at pos {pos}: existing {hex(old)} vs new {hex(new_k)} (hint {ci},{pos},{ch})")
    key[pos] = new_k


Kiểm tra printable khi chọn candidate key:

def is_printable_byte(b):
    return 32 <= b <= 126
# trong fill, chỉ count candidate nếu ct[k][i] ^ candidate là printable.


Kịch bản “chỉ threshold” để so sánh

Bạn đã yêu cầu: bỏ fill để thấy tác dụng threshold; giữ code như phiên bản bạn có với threshold=5/7/9 và so sánh.

Sử dụng logging / export key:

import json
with open('partial_key.json','w') as f:
    json.dump({str(k): key[k] for k in key}, f)


Không dùng target làm hint — chỉ dùng các ciphertext khác để refine.

Hướng dẫn để viết một báo cáo (mẫu ngắn) — bạn có thể copy/paste

(để đưa vào nộp bài hoặc trình bày)

Tiêu đề: Giải mã bộ ciphertext khi reuse keystream (OTP reuse) — báo cáo

Mục tiêu: Khôi phục plaintext (target) từ tập ciphertext (11 bản) chia sẻ cùng một keystream.

Phương pháp:

Dùng quan hệ XOR giữa mọi cặp ciphertext: ct_i ⊕ ct_j = pt_i ⊕ pt_j.

Ứng dụng heuristic: nếu ct_i ⊕ ct_j tại pos là chữ, thì khả năng pt_i[pos] hoặc pt_j[pos] là space; dùng voting trên cặp để phát hiện vị trí space.

Đặt threshold để kiểm soát tỉ lệ false positives.

Sinh partial_key từ các vị trí được vote; decrypt các ciphertext với partial_key.

(Tùy chọn) fill thêm vị trí chưa biết bằng phương pháp giả định “space” trên từng ciphertext và chọn candidate nhận được nhiều đồng thuận nhất.

(Tùy chọn) apply manual hints tính toán từ plaintext known → tính chính xác key byte.

Kết quả:

Khi dùng chỉ threshold (khá bảo thủ) ta thu được phần plaintext có độ tin cậy cao nhưng còn nhiều ?.

Khi thêm fill heuristic kết hợp vài manual hints từ ciphertext (không target) ta lấp được nhiều vị trí, target gần đúng.

Nếu dùng manual hints trực tiếp từ target (tức đã biết plaintext), ta có thể khôi phục target chính xác — nhưng đó là “đưa đáp án vào” hơn là tấn công tự động.

Phân tích sai sót:

Những vị trí không có enough evidence (ví dụ “using a stream” xuất hiện ít) thường không được recover tự động.

Fill heuristic mạnh nhưng dễ tạo false positives.

Kết luận / Khuyến nghị:

Không reuse keystream! Reuse phá tính bảo mật của stream cipher.

Khi tấn công: phối hợp threshold (seed) + fill (explore) + manual hints từ non-target ciphertexts + (tuỳ chọn) dictionary/n-gram refine.

Một checklist bạn có thể dùng để debug nhanh

 Chạy analyze_threshold với threshold = {5,6,7,8} → ghi lại số vị trí key được tìm.

 Chạy decrypt chỉ với threshold (không fill) → lưu output.

 Nếu muốn lấp: chạy fill chỉ cho pos chưa có key (không ghi đè).

 Liệt kê các vị trí key được chọn bởi fill có best_count thấp (ví dụ < 3) — đánh dấu là “nghi ngờ”.

 Áp manual hint chỉ trên ciphertext khác (không target).

 Kiểm tra conflicts trước khi ghi key từ hint.

 Lưu key/outputs để tái lập.

Kết luận nhanh (tóm tắt)

Code hoạt động theo một heuristic hợp lệ cho kịch bản keystream reuse: seed bằng vote dựa trên ct_i ⊕ ct_j → suy space → tính key.

threshold có ý nghĩa khi bạn không dùng fill mạnh; khi dùng fill nó trở nên ít quan trọng.

fill_key_with_space_trick giúp lấp ? nhưng gây sai — trade-off giữa coverage và accuracy.

manual_hints là phép toán (XOR) chắc chắn: nó không phải đoán mò — nhưng hãy tránh dùng target làm hint nếu bạn muốn tấn công “thực sự tự động”.

Để sửa lỗi như usi|g wsstream bạn cần thêm bằng chứng (ciphertexts khác) hoặc một vài manual hints từ ciphertext khác, không phải từ target.
