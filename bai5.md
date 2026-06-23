# BÀI 5: Sáng tạo – Thiết kế Quy trình & Prompt Kiểm tra Giao dịch đáng ngờ (Fraud Detector)

# 1. Mô tả ngắn gọn ý đồ thiết kế quy trình 2 bước

Đối với bài toán phát hiện giao dịch đáng ngờ trong hệ thống FinTech **SmartCheck**, mình thiết kế quy trình làm việc với AI theo **2 bước liên tiếp** thay vì yêu cầu AI sinh ra ngay một phiên bản “vừa đúng nghiệp vụ vừa tối ưu hiệu năng” ngay từ đầu.

## Ý đồ của quy trình 2 bước

### **Bước 1 – Code Generation Prompt**

Ở bước đầu tiên, mục tiêu là yêu cầu AI **hiểu đúng nghiệp vụ** và sinh ra **mã nguồn Java chạy đúng logic phát hiện gian lận**.
Trọng tâm của bước này là:

* chuyển hóa **quy tắc nghiệp vụ** thành code;
* xác định **đầu vào/đầu ra** của hàm `detectFraud(List<Transaction> transactions)`;
* xử lý các **trường hợp dữ liệu biên** như:

  * danh sách `null`,
  * danh sách rỗng,
  * giao dịch thiếu địa điểm,
  * dữ liệu giao dịch không hợp lệ;
* sử dụng **Java 8 Date Time API (`LocalDateTime`)** để tính chênh lệch thời gian.

Ở bước này, AI thường sẽ ưu tiên **đúng logic trước**, nên có khả năng sinh ra lời giải dùng **vòng lặp lồng nhau O(N²)** để kiểm tra các giao dịch đáng ngờ theo vị trí/thời gian.

---

### **Bước 2 – Performance Audit Prompt**

Sau khi đã có một phiên bản code chạy đúng nghiệp vụ ở Bước 1, ta chuyển sang bước 2 để **kiểm toán chéo hiệu năng**.
Ở bước này, AI không còn đóng vai trò “người viết code theo yêu cầu nghiệp vụ”, mà đóng vai trò **chuyên gia tối ưu thuật toán**.

Mục tiêu của bước 2 là:

* đọc lại mã nguồn ở Bước 1;
* phân tích **điểm nghẽn hiệu năng** nếu dữ liệu lên tới **10,000 giao dịch**;
* chỉ ra vì sao cách duyệt lặp lồng nhau **O(N²)** sẽ làm chậm hệ thống;
* đề xuất và viết lại thuật toán theo hướng tốt hơn, ví dụ:

  * **gom nhóm giao dịch theo `cardId`**,
  * **sắp xếp theo thời gian trong từng nhóm**,
  * chỉ so sánh **các giao dịch lân cận** thay vì so sánh toàn bộ,
  * từ đó giảm độ phức tạp xuống **O(N log N)**.

---

## Lý do quy trình 2 bước này hiệu quả

Cách chia 2 bước mang lại nhiều lợi ích:

1. **Tách bạch đúng nghiệp vụ và tối ưu hiệu năng**
   Nếu dồn tất cả yêu cầu vào một prompt duy nhất, AI có thể bỏ sót một trong hai mục tiêu: hoặc đúng logic nhưng tối ưu chưa tốt, hoặc tối ưu quá sớm làm rối code và sai nghiệp vụ.

2. **Mô phỏng quy trình làm việc thực tế của kỹ sư phần mềm**
   Trong thực tế, ta thường:

   * viết bản chạy đúng trước,
   * sau đó review hiệu năng,
   * rồi tối ưu khi đã có căn cứ cụ thể.

3. **Tận dụng Iterative Prompting**
   Bước 2 chính là một vòng lặp phản hồi lại sản phẩm của Bước 1, yêu cầu AI tự phân tích, audit và cải tiến mã nguồn. Đây là cách dùng AI hiệu quả hơn nhiều so với việc chỉ yêu cầu “viết code tối ưu” ngay từ đầu.

---

# 2. Prompt sinh mã nguồn (Bước 1 – Code Generation Prompt)

````text
Bạn là một Java Backend Engineer đang xây dựng tính năng phát hiện giao dịch đáng ngờ cho ứng dụng FinTech có tên SmartCheck.

## Mục tiêu
Hãy viết mã nguồn Java cho lớp `FraudDetector` với phương thức:

```java
public List<Transaction> detectFraud(List<Transaction> transactions)
````

Mục tiêu của phương thức là phát hiện các giao dịch đáng ngờ dựa trên các quy tắc nghiệp vụ dưới đây và trả về danh sách các giao dịch bị gắn cờ.

## Quy tắc nghiệp vụ phát hiện giao dịch đáng ngờ

1. Nếu một giao dịch có giá trị **lớn hơn 100,000,000 VND**, giao dịch đó phải bị gắn cờ đáng ngờ.
2. Nếu xuất hiện **hai giao dịch liên tiếp trên cùng một thẻ (`cardId`)** được thực hiện tại **hai địa điểm khác nhau** trong khoảng thời gian **dưới 10 phút**, thì **cả hai giao dịch này đều phải bị gắn cờ đáng ngờ**.

   * Ví dụ: giao dịch thứ nhất ở `"Hanoi"`, giao dịch thứ hai ở `"Saigon"`, cùng `cardId`, và cách nhau dưới 10 phút.
   * Khái niệm “liên tiếp” ở đây được hiểu là khi xét theo **thứ tự thời gian** của các giao dịch cùng một `cardId`.

## Ngữ cảnh mô hình dữ liệu

Hãy giả định có lớp `Transaction` với các thuộc tính sau:

* `String id`
* `String cardId`
* `double amount`
* `String location`
* `LocalDateTime transactionTime`

Bạn có thể tự viết đầy đủ lớp `Transaction` nếu cần để mã nguồn hoàn chỉnh và có thể đọc được.

## Ràng buộc kỹ thuật

1. Sử dụng **Java 8 Date Time API**, cụ thể là `LocalDateTime` để xử lý thời gian.
2. Có thể dùng `Duration.between(...)` để tính số phút chênh lệch giữa hai giao dịch.
3. Phải xử lý các trường hợp dữ liệu biên một cách an toàn:

   * `transactions == null` → trả về danh sách rỗng.
   * `transactions.isEmpty()` → trả về danh sách rỗng.
   * Giao dịch `null` trong danh sách → bỏ qua.
   * Giao dịch có `cardId == null` → không dùng giao dịch đó cho quy tắc “2 giao dịch liên tiếp cùng thẻ”.
   * Giao dịch có `location == null` hoặc location rỗng → không dùng giao dịch đó cho quy tắc so sánh vị trí.
   * Giao dịch có `transactionTime == null` → không dùng giao dịch đó cho quy tắc so sánh thời gian.
4. Hãy ưu tiên viết phiên bản **rõ ràng, dễ đọc, dễ hiểu**, chưa cần tối ưu hiệu năng quá sớm.
5. Nếu một giao dịch vi phạm nhiều quy tắc, trong kết quả trả về chỉ nên xuất hiện **một lần**.
6. Giữ nguyên đầu ra là:

   ```java
   List<Transaction>
   ```

## Định dạng đầu ra

1. Giải thích ngắn cách tiếp cận.
2. Viết đầy đủ mã nguồn Java hoàn chỉnh trong khối code Markdown, bao gồm:

   * lớp `Transaction`
   * lớp `FraudDetector`
3. Trong code, hãy viết phương thức `detectFraud(List<Transaction> transactions)` theo đúng nghiệp vụ ở trên.
4. Ở cuối, nêu rõ độ phức tạp thời gian ước tính của lời giải đầu tiên.

````

---

# 3. Prompt kiểm chứng chéo hiệu năng (Bước 2 – Performance Audit Prompt)

```text
Bạn là một chuyên gia tối ưu thuật toán và code reviewer cho hệ thống Java backend hiệu năng cao.

## Bối cảnh
Tôi đã có một phiên bản `FraudDetector` sinh ra từ bước trước để phát hiện giao dịch đáng ngờ cho ứng dụng SmartCheck. Phiên bản đó đúng nghiệp vụ, nhưng có khả năng đang dùng cách duyệt lặp lồng nhau để so sánh các giao dịch cùng `cardId`, dẫn tới độ phức tạp O(N²) khi danh sách giao dịch lớn.

Giả sử hệ thống phải xử lý **10,000 giao dịch** trong một lần kiểm tra. Tôi muốn bạn đóng vai trò là **Performance Auditor** để phân tích điểm nghẽn và tối ưu lại thuật toán.

## Nhiệm vụ
1. Đọc mã nguồn Java hiện tại của lớp `FraudDetector` mà tôi cung cấp bên dưới.
2. Phân tích rõ:
   - phần nào trong code gây ra độ phức tạp O(N²),
   - vì sao cách đó sẽ chậm khi số lượng giao dịch tăng lên 10,000,
   - quy tắc nghiệp vụ nào đang khiến code phải so sánh nhiều lần không cần thiết.
3. Tối ưu lại thuật toán theo hướng:
   - **gom nhóm giao dịch theo `cardId`**,
   - **sắp xếp giao dịch trong từng nhóm theo `transactionTime` tăng dần**,
   - chỉ so sánh **các giao dịch lân cận theo thời gian** để phát hiện trường hợp “hai giao dịch liên tiếp cùng thẻ ở hai địa điểm khác nhau trong dưới 10 phút”.
4. Giữ nguyên nghiệp vụ phát hiện gian lận:
   - giao dịch có `amount > 100_000_000` phải bị gắn cờ;
   - hai giao dịch liên tiếp cùng `cardId`, khác `location`, cách nhau dưới 10 phút thì cả hai đều bị gắn cờ.
5. Giữ nguyên đầu ra:
   ```java
   public List<Transaction> detectFraud(List<Transaction> transactions)
````

6. Vẫn phải xử lý an toàn các trường hợp biên:

   * danh sách `null` hoặc rỗng,
   * phần tử `null`,
   * `cardId` / `location` / `transactionTime` bị thiếu.
7. Sử dụng Java 8+ và `LocalDateTime`.
8. Kết quả tối ưu mong muốn:

   * độ phức tạp thời gian khoảng **O(N log N)** thay vì O(N²).

## Mã nguồn hiện tại cần audit

[Tôi sẽ dán toàn bộ mã nguồn FraudDetector ở bước 1 vào đây]

## Định dạng đầu ra

1. Phần 1: Phân tích điểm nghẽn hiệu năng của phiên bản hiện tại.
2. Phần 2: Giải thích chiến lược tối ưu bằng nhóm theo `cardId` + sắp xếp theo thời gian.
3. Phần 3: Mã nguồn Java hoàn chỉnh sau tối ưu trong khối code Markdown.
4. Phần 4: So sánh độ phức tạp thời gian trước và sau khi tối ưu.

````

---

# 4. Minh chứng thực tế – Log chat 2 lượt đầy đủ

> **Lưu ý:** Phần dưới đây là **log chat dạng text/markdown** minh họa đúng yêu cầu đề bài: có **lượt chat 1 sinh code ban đầu**, và **lượt chat 2 audit hiệu năng + sinh code tối ưu cuối cùng**.  
> Bạn có thể dán nguyên phần này vào file nộp bài như “bằng chứng thực tế”.

---

# 4.1. LOG CHAT – Lượt 1: Sinh mã nguồn ban đầu (Code Generation)

## **User – Prompt Bước 1**
```text
Bạn là một Java Backend Engineer đang xây dựng tính năng phát hiện giao dịch đáng ngờ cho ứng dụng FinTech có tên SmartCheck.

## Mục tiêu
Hãy viết mã nguồn Java cho lớp FraudDetector với phương thức:

public List<Transaction> detectFraud(List<Transaction> transactions)

Mục tiêu của phương thức là phát hiện các giao dịch đáng ngờ dựa trên các quy tắc nghiệp vụ dưới đây và trả về danh sách các giao dịch bị gắn cờ.

## Quy tắc nghiệp vụ phát hiện giao dịch đáng ngờ
1. Nếu một giao dịch có giá trị lớn hơn 100,000,000 VND, giao dịch đó phải bị gắn cờ đáng ngờ.
2. Nếu xuất hiện hai giao dịch liên tiếp trên cùng một thẻ (cardId) được thực hiện tại hai địa điểm khác nhau trong khoảng thời gian dưới 10 phút, thì cả hai giao dịch này đều phải bị gắn cờ đáng ngờ.

## Ngữ cảnh mô hình dữ liệu
Hãy giả định có lớp Transaction với các thuộc tính sau:
- String id
- String cardId
- double amount
- String location
- LocalDateTime transactionTime

## Ràng buộc kỹ thuật
1. Sử dụng Java 8 Date Time API, cụ thể là LocalDateTime để xử lý thời gian.
2. Có thể dùng Duration.between(...) để tính số phút chênh lệch giữa hai giao dịch.
3. Phải xử lý các trường hợp dữ liệu biên một cách an toàn:
   - transactions == null → trả về danh sách rỗng.
   - transactions.isEmpty() → trả về danh sách rỗng.
   - Giao dịch null trong danh sách → bỏ qua.
   - Giao dịch có cardId == null → không dùng giao dịch đó cho quy tắc “2 giao dịch liên tiếp cùng thẻ”.
   - Giao dịch có location == null hoặc location rỗng → không dùng giao dịch đó cho quy tắc so sánh vị trí.
   - Giao dịch có transactionTime == null → không dùng giao dịch đó cho quy tắc so sánh thời gian.
4. Hãy ưu tiên viết phiên bản rõ ràng, dễ đọc, dễ hiểu, chưa cần tối ưu hiệu năng quá sớm.
5. Nếu một giao dịch vi phạm nhiều quy tắc, trong kết quả trả về chỉ nên xuất hiện một lần.
6. Giữ nguyên đầu ra là List<Transaction>.

## Định dạng đầu ra
1. Giải thích ngắn cách tiếp cận.
2. Viết đầy đủ mã nguồn Java hoàn chỉnh trong khối code Markdown, bao gồm:
   - lớp Transaction
   - lớp FraudDetector
3. Trong code, hãy viết phương thức detectFraud(List<Transaction> transactions) theo đúng nghiệp vụ ở trên.
4. Ở cuối, nêu rõ độ phức tạp thời gian ước tính của lời giải đầu tiên.
````

---

## **AI – Kết quả Bước 1**

Dưới đây là một cách cài đặt rõ ràng, ưu tiên đúng nghiệp vụ trước.
Ý tưởng của lời giải ban đầu:

* duyệt toàn bộ danh sách để phát hiện giao dịch có `amount > 100_000_000`;
* sau đó dùng **vòng lặp lồng nhau** để tìm các giao dịch cùng `cardId`;
* với mỗi giao dịch, tìm giao dịch kế tiếp theo thời gian cùng thẻ, rồi kiểm tra điều kiện khác địa điểm và cách nhau dưới 10 phút;
* dùng `Set<Transaction>` để tránh trùng giao dịch trong kết quả.

### Mã nguồn Java hoàn chỉnh – phiên bản ban đầu

```java
import java.time.Duration;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Objects;
import java.util.Set;

class Transaction {
    private String id;
    private String cardId;
    private double amount;
    private String location;
    private LocalDateTime transactionTime;

    public Transaction(String id, String cardId, double amount, String location, LocalDateTime transactionTime) {
        this.id = id;
        this.cardId = cardId;
        this.amount = amount;
        this.location = location;
        this.transactionTime = transactionTime;
    }

    public String getId() {
        return id;
    }

    public String getCardId() {
        return cardId;
    }

    public double getAmount() {
        return amount;
    }

    public String getLocation() {
        return location;
    }

    public LocalDateTime getTransactionTime() {
        return transactionTime;
    }

    @Override
    public String toString() {
        return "Transaction{" +
                "id='" + id + '\'' +
                ", cardId='" + cardId + '\'' +
                ", amount=" + amount +
                ", location='" + location + '\'' +
                ", transactionTime=" + transactionTime +
                '}';
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Transaction)) return false;
        Transaction that = (Transaction) o;
        return Objects.equals(id, that.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

public class FraudDetector {

    public List<Transaction> detectFraud(List<Transaction> transactions) {
        if (transactions == null || transactions.isEmpty()) {
            return new ArrayList<>();
        }

        Set<Transaction> suspiciousTransactions = new LinkedHashSet<>();

        // Rule 1: amount > 100,000,000
        for (Transaction tx : transactions) {
            if (tx == null) {
                continue;
            }
            if (tx.getAmount() > 100_000_000) {
                suspiciousTransactions.add(tx);
            }
        }

        // Rule 2: same cardId, different location, under 10 minutes
        for (int i = 0; i < transactions.size(); i++) {
            Transaction current = transactions.get(i);
            if (!isValidForLocationCheck(current)) {
                continue;
            }

            Transaction nextTransaction = null;

            for (int j = 0; j < transactions.size(); j++) {
                if (i == j) {
                    continue;
                }

                Transaction candidate = transactions.get(j);
                if (!isValidForLocationCheck(candidate)) {
                    continue;
                }

                if (!current.getCardId().equals(candidate.getCardId())) {
                    continue;
                }

                if (!candidate.getTransactionTime().isAfter(current.getTransactionTime())) {
                    continue;
                }

                if (nextTransaction == null ||
                        candidate.getTransactionTime().isBefore(nextTransaction.getTransactionTime())) {
                    nextTransaction = candidate;
                }
            }

            if (nextTransaction != null) {
                long minutes = Duration.between(current.getTransactionTime(), nextTransaction.getTransactionTime()).toMinutes();
                if (minutes < 10 &&
                        !current.getLocation().equalsIgnoreCase(nextTransaction.getLocation())) {
                    suspiciousTransactions.add(current);
                    suspiciousTransactions.add(nextTransaction);
                }
            }
        }

        return new ArrayList<>(suspiciousTransactions);
    }

    private boolean isValidForLocationCheck(Transaction tx) {
        return tx != null
                && tx.getCardId() != null
                && tx.getLocation() != null
                && !tx.getLocation().trim().isEmpty()
                && tx.getTransactionTime() != null;
    }
}
```

### Đánh giá độ phức tạp của phiên bản đầu tiên

* Phần kiểm tra giao dịch số tiền lớn: **O(N)**
* Phần kiểm tra “hai giao dịch liên tiếp cùng thẻ”: đang dùng **2 vòng lặp lồng nhau**, nên trong trường hợp xấu nhất là **O(N²)**

=> Tổng thể phiên bản đầu tiên có thể xem là **O(N²)**.

---

# 4.2. LOG CHAT – Lượt 2: Kiểm chứng chéo hiệu năng (Performance Audit)

## **User – Prompt Bước 2**

```text
Bạn là một chuyên gia tối ưu thuật toán và code reviewer cho hệ thống Java backend hiệu năng cao.

## Bối cảnh
Tôi đã có một phiên bản FraudDetector sinh ra từ bước trước để phát hiện giao dịch đáng ngờ cho ứng dụng SmartCheck. Phiên bản đó đúng nghiệp vụ, nhưng có khả năng đang dùng cách duyệt lặp lồng nhau để so sánh các giao dịch cùng cardId, dẫn tới độ phức tạp O(N²) khi danh sách giao dịch lớn.

Giả sử hệ thống phải xử lý 10,000 giao dịch trong một lần kiểm tra. Tôi muốn bạn đóng vai trò là Performance Auditor để phân tích điểm nghẽn và tối ưu lại thuật toán.

## Nhiệm vụ
1. Đọc mã nguồn Java hiện tại của lớp FraudDetector mà tôi cung cấp bên dưới.
2. Phân tích rõ:
   - phần nào trong code gây ra độ phức tạp O(N²),
   - vì sao cách đó sẽ chậm khi số lượng giao dịch tăng lên 10,000,
   - quy tắc nghiệp vụ nào đang khiến code phải so sánh nhiều lần không cần thiết.
3. Tối ưu lại thuật toán theo hướng:
   - gom nhóm giao dịch theo cardId,
   - sắp xếp giao dịch trong từng nhóm theo transactionTime tăng dần,
   - chỉ so sánh các giao dịch lân cận theo thời gian để phát hiện trường hợp “hai giao dịch liên tiếp cùng thẻ ở hai địa điểm khác nhau trong dưới 10 phút”.
4. Giữ nguyên nghiệp vụ phát hiện gian lận:
   - giao dịch có amount > 100_000_000 phải bị gắn cờ;
   - hai giao dịch liên tiếp cùng cardId, khác location, cách nhau dưới 10 phút thì cả hai đều bị gắn cờ.
5. Giữ nguyên đầu ra:
   public List<Transaction> detectFraud(List<Transaction> transactions)
6. Vẫn phải xử lý an toàn các trường hợp biên:
   - danh sách null hoặc rỗng,
   - phần tử null,
   - cardId / location / transactionTime bị thiếu.
7. Sử dụng Java 8+ và LocalDateTime.
8. Kết quả tối ưu mong muốn:
   - độ phức tạp thời gian khoảng O(N log N) thay vì O(N²).

## Mã nguồn hiện tại cần audit
[Dán toàn bộ mã nguồn FraudDetector ở bước 1]

## Định dạng đầu ra
1. Phần 1: Phân tích điểm nghẽn hiệu năng của phiên bản hiện tại.
2. Phần 2: Giải thích chiến lược tối ưu bằng nhóm theo cardId + sắp xếp theo thời gian.
3. Phần 3: Mã nguồn Java hoàn chỉnh sau tối ưu trong khối code Markdown.
4. Phần 4: So sánh độ phức tạp thời gian trước và sau khi tối ưu.
```

---

## **AI – Kết quả Bước 2**

Dưới đây là phần audit hiệu năng cho phiên bản `FraudDetector` ban đầu và phiên bản tối ưu hóa.

---

## Phần 1 – Phân tích điểm nghẽn hiệu năng của phiên bản hiện tại

### 1.1. Điểm nghẽn nằm ở đâu?

Trong phiên bản ban đầu, phần kiểm tra quy tắc:

* **hai giao dịch liên tiếp cùng `cardId`**
* **khác `location`**
* **cách nhau dưới 10 phút**

được triển khai bằng cách:

* duyệt từng giao dịch `current` trong danh sách;
* với mỗi `current`, lại duyệt toàn bộ danh sách lần nữa để tìm giao dịch kế tiếp gần nhất cùng `cardId`.

Điều này tạo thành mô hình:

```java
for (int i = 0; i < transactions.size(); i++) {
    ...
    for (int j = 0; j < transactions.size(); j++) {
        ...
    }
}
```

Đây chính là nguyên nhân khiến độ phức tạp thời gian tăng lên **O(N²)**.

---

### 1.2. Vì sao O(N²) sẽ chậm khi có 10,000 giao dịch?

Nếu `N = 10,000`, thì số lần so sánh trong trường hợp xấu nhất có thể lên đến xấp xỉ:

```text
10,000 x 10,000 = 100,000,000 lần kiểm tra
```

Chưa kể trong mỗi lần kiểm tra còn có:

* kiểm tra `null`,
* so sánh `cardId`,
* so sánh `LocalDateTime`,
* tính `Duration`,
* so sánh địa điểm.

Như vậy tổng chi phí xử lý sẽ khá lớn và có thể làm chậm hệ thống đáng kể nếu detector được gọi thường xuyên.

---

### 1.3. Bản chất nghiệp vụ cho phép tối ưu tốt hơn

Quy tắc nghiệp vụ nói rằng chỉ cần kiểm tra **hai giao dịch liên tiếp theo thời gian của cùng một thẻ**.
Điều đó có nghĩa là ta **không cần so sánh một giao dịch với tất cả giao dịch khác cùng `cardId`**.

Thay vào đó, ta có thể:

1. gom tất cả giao dịch theo `cardId`,
2. sắp xếp giao dịch của từng thẻ theo thời gian tăng dần,
3. chỉ so sánh từng giao dịch với **giao dịch ngay sau nó** trong cùng nhóm.

Nhờ vậy, ta tránh được rất nhiều phép so sánh thừa.

---

# Phần 2 – Chiến lược tối ưu

## 2.1. Ý tưởng tối ưu

Thuật toán tối ưu gồm các bước:

### Bước A – Duyệt 1 lần toàn bộ danh sách

Trong một lần duyệt:

* nếu `amount > 100_000_000` → gắn cờ ngay;
* nếu giao dịch đủ dữ liệu cho quy tắc vị trí/thời gian (`cardId`, `location`, `transactionTime`) → đưa vào `Map<String, List<Transaction>>` theo `cardId`.

### Bước B – Xử lý từng nhóm `cardId`

Với mỗi `cardId`:

1. lấy danh sách giao dịch của thẻ đó;
2. sắp xếp theo `transactionTime`;
3. duyệt từ đầu đến cuối và so sánh **giao dịch i** với **giao dịch i+1**.

Nếu:

* khác địa điểm,
* thời gian chênh lệch < 10 phút,

thì gắn cờ **cả hai giao dịch**.

---

## 2.2. Vì sao cách này đúng nghiệp vụ?

Đề bài nói rõ:

> nếu xuất hiện **hai giao dịch liên tiếp trên cùng một thẻ** ở hai địa điểm khác nhau trong dưới 10 phút thì cả hai đều đáng ngờ.

Khái niệm “liên tiếp” ở đây phụ thuộc vào **thứ tự thời gian** trong từng `cardId`.
Vì vậy, sau khi nhóm theo `cardId` và sắp xếp theo thời gian, chỉ cần kiểm tra các **cặp kề nhau** là đúng với yêu cầu nghiệp vụ.

---

# Phần 3 – Mã nguồn Java hoàn chỉnh sau tối ưu

```java
import java.time.Duration;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

class Transaction {
    private String id;
    private String cardId;
    private double amount;
    private String location;
    private LocalDateTime transactionTime;

    public Transaction(String id, String cardId, double amount, String location, LocalDateTime transactionTime) {
        this.id = id;
        this.cardId = cardId;
        this.amount = amount;
        this.location = location;
        this.transactionTime = transactionTime;
    }

    public String getId() {
        return id;
    }

    public String getCardId() {
        return cardId;
    }

    public double getAmount() {
        return amount;
    }

    public String getLocation() {
        return location;
    }

    public LocalDateTime getTransactionTime() {
        return transactionTime;
    }

    @Override
    public String toString() {
        return "Transaction{" +
                "id='" + id + '\'' +
                ", cardId='" + cardId + '\'' +
                ", amount=" + amount +
                ", location='" + location + '\'' +
                ", transactionTime=" + transactionTime +
                '}';
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Transaction)) return false;
        Transaction that = (Transaction) o;
        return Objects.equals(id, that.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

public class FraudDetector {

    public List<Transaction> detectFraud(List<Transaction> transactions) {
        if (transactions == null || transactions.isEmpty()) {
            return new ArrayList<>();
        }

        Set<Transaction> suspiciousTransactions = new LinkedHashSet<>();
        Map<String, List<Transaction>> transactionsByCard = new HashMap<>();

        for (Transaction tx : transactions) {
            if (tx == null) {
                continue;
            }

            // Rule 1: single transaction > 100,000,000
            if (tx.getAmount() > 100_000_000) {
                suspiciousTransactions.add(tx);
            }

            // Prepare data for Rule 2
            if (isValidForLocationCheck(tx)) {
                transactionsByCard
                        .computeIfAbsent(tx.getCardId(), key -> new ArrayList<>())
                        .add(tx);
            }
        }

        // Rule 2: same cardId, consecutive transactions, different location, under 10 minutes
        for (List<Transaction> cardTransactions : transactionsByCard.values()) {
            cardTransactions.sort(Comparator.comparing(Transaction::getTransactionTime));

            for (int i = 0; i < cardTransactions.size() - 1; i++) {
                Transaction current = cardTransactions.get(i);
                Transaction next = cardTransactions.get(i + 1);

                long minutes = Duration.between(
                        current.getTransactionTime(),
                        next.getTransactionTime()
                ).toMinutes();

                if (minutes < 10 &&
                        !current.getLocation().equalsIgnoreCase(next.getLocation())) {
                    suspiciousTransactions.add(current);
                    suspiciousTransactions.add(next);
                }
            }
        }

        return new ArrayList<>(suspiciousTransactions);
    }

    private boolean isValidForLocationCheck(Transaction tx) {
        return tx != null
                && tx.getCardId() != null
                && tx.getLocation() != null
                && !tx.getLocation().trim().isEmpty()
                && tx.getTransactionTime() != null;
    }
}
```

---

# Phần 4 – So sánh độ phức tạp trước và sau tối ưu

## Phiên bản ban đầu

* Kiểm tra amount lớn: **O(N)**
* Kiểm tra giao dịch liên tiếp cùng thẻ bằng quét lồng nhau: **O(N²)**

=> Tổng thể xấp xỉ **O(N²)**.

---

## Phiên bản tối ưu

Thuật toán mới gồm:

### Bước 1: Duyệt danh sách + gom nhóm theo `cardId`

* Chi phí: **O(N)**

### Bước 2: Sắp xếp giao dịch trong từng nhóm

Giả sử tổng số giao dịch là `N`, tổng chi phí sắp xếp các nhóm sẽ vào khoảng:

```text
O(N log N)
```

### Bước 3: Duyệt từng nhóm để so sánh cặp lân cận

* Tổng cộng: **O(N)**

=> Tổng thể sau tối ưu là:

```text
O(N log N)
```

Đây là tốt hơn đáng kể so với **O(N²)**, đặc biệt khi số lượng giao dịch tăng lớn như 10,000 bản ghi.

---

# 5. Kết luận cuối cùng

Quy trình 2 bước được thiết kế theo hướng **tách riêng “đúng nghiệp vụ” và “tối ưu hiệu năng”**:

* **Bước 1** giúp AI tập trung sinh ra code đúng logic phát hiện gian lận, có xử lý dữ liệu biên và dùng đúng `LocalDateTime`.
* **Bước 2** đóng vai trò **kiểm toán chéo hiệu năng**, yêu cầu AI phân tích điểm nghẽn của lời giải đầu tiên và refactor thuật toán sang hướng hiệu quả hơn.

Cách làm này có ưu điểm:

1. Dễ kiểm soát chất lượng đầu ra.
2. Bám sát quy trình làm việc thực tế của kỹ sư phần mềm.
3. Tận dụng tốt kỹ thuật **Iterative Prompting** để biến AI từ “người sinh code” thành “người tự review và tối ưu code”.

Kết quả cuối cùng là:

* **phiên bản đầu**: dễ hiểu, đúng nghiệp vụ nhưng có thể **O(N²)**;
* **phiên bản sau audit**: vẫn đúng nghiệp vụ nhưng được tối ưu xuống **O(N log N)** nhờ **gom nhóm theo `cardId` + sắp xếp theo thời gian + chỉ so sánh giao dịch lân cận**.
