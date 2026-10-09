# [Thực hành] JDBC Transaction – Sử dụng Transaction (All-or-Nothing)

## Mục tiêu:

Hiểu và vận dụng thành thạo JDBC Transaction để kiểm soát quá trình ghi dữ liệu, đảm bảo nguyên lý All-or-Nothing (Thành công tất cả, hoặc không có gì cả).

## Mô tả:

JDBC Transaction đảm bảo một tập các câu lệnh SQL được thực thi như một đơn vị duy nhất.

* **Trường hợp 1:** Tất cả các câu lệnh SQL đều thực thi thành công. Dữ liệu được ghi nhận vĩnh viễn (Commit).
* **Trường hợp 2:** Có bất kỳ câu lệnh nào thất bại. Trạng thái cơ sở dữ liệu sẽ được khôi phục lại y hệt như thời điểm trước khi thực thi (Rollback).

Trong ví dụ này, chúng ta sẽ tiếp tục chạy lại kịch bản chèn 2 hàng và cập nhật 1 hàng trên bảng Employee nhưng **đã bật Transaction**.

## Hướng dẫn:

### Bước 1: Cập nhật Interface IUserDAO

*(Lưu ý: Nếu bạn đang làm tiếp nối bài trước, bảng Employee đã được tạo sẵn trong MySQL. Bạn không cần tạo lại bảng nữa).*

Mở file `IUserDAO.java` và bổ sung thêm phương thức xử lý có dùng Transaction:

```java
public void insertUpdateUseTransaction() throws SQLException;
```

### Bước 2: Cập nhật lớp UserDAO

Mở file `UserDAO.java`. Lần này, chúng ta sẽ triển khai phương thức `insertUpdateUseTransaction()` với điểm nhấn quan trọng là điều khiển cờ AutoCommit.

```java
@Override
public void insertUpdateUseTransaction() {
    try (Connection conn = getConnection();
         Statement statement = conn.createStatement();
         PreparedStatement psInsert = conn.prepareStatement(SQL_INSERT);
         PreparedStatement psUpdate = conn.prepareStatement(SQL_UPDATE)) {

        statement.execute(SQL_TABLE_DROP);
        statement.execute(SQL_TABLE_CREATE);

        // ==========================================
        // 1. BẮT ĐẦU TRANSACTION BLOCK
        // Tắt chế độ lưu tự động (mặc định là true)
        // ==========================================
        conn.setAutoCommit(false); 

        // 2. Chạy danh sách lệnh Insert
        psInsert.setString(1, "Quynh");
        psInsert.setBigDecimal(2, new BigDecimal(10));
        psInsert.setTimestamp(3, Timestamp.valueOf(LocalDateTime.now()));
        psInsert.execute();

        psInsert.setString(1, "Ngan");
        psInsert.setBigDecimal(2, new BigDecimal(20));
        psInsert.setTimestamp(3, Timestamp.valueOf(LocalDateTime.now()));
        psInsert.execute();

        // 3. Chạy lệnh Update (Cố tình tạo lỗi để test)
        // Dòng bên dưới sẽ gây lỗi: org.mysqlsql.util.PSQLException: No value specified...
        psUpdate.setBigDecimal(2, new BigDecimal(999.99));
        // Đáng lẽ phải là: psUpdate.setBigDecimal(1, new BigDecimal(999.99));
        
        psUpdate.setString(2, "Quynh");
        psUpdate.execute();

        // ==========================================
        // 4. KẾT THÚC TRANSACTION BLOCK
        // Nếu code chạy trót lọt đến đây, tiến hành ghi dữ liệu
        // ==========================================
        conn.commit();
        
        // Thói quen tốt: Bật lại chế độ auto-commit về mặc định
        conn.setAutoCommit(true);

    } catch (Exception e) {
        System.out.println("Lỗi xảy ra, Transaction sẽ tự động huỷ bỏ (rollback) khi đóng kết nối!");
        System.out.println(e.getMessage());
        e.printStackTrace();
    }
}
```

Việc gọi `conn.setAutoCommit(false)` chính là chiếc công tắc kích hoạt Transaction. Nếu luồng mã bị văng lỗi và rơi vào khối catch, lệnh `conn.commit()` sẽ không bao giờ được gọi. Khi cấu trúc try-with-resources tự động đóng kết nối (close), JDBC sẽ huỷ bỏ toàn bộ các lệnh Insert chưa được commit trước đó.

### Bước 3: Cập nhật UserServlet

Mở file `UserServlet.java` để bổ sung định tuyến URL cho chức năng mới.

Bổ sung case mới vào khối switch trong phương thức `doGet()`:

```java
case "test-use-tran":
    testUseTran(request, response);
    break;
```

Viết thêm phương thức `testUseTran()` để gọi hàm xử lý từ DAO:

```java
private void testUseTran(HttpServletRequest request, HttpServletResponse response) {
    userDAO.insertUpdateUseTransaction();
    System.out.println("Hoàn tất gọi hàm testUseTran!");
}
```

### Bước 4: Chạy ứng dụng và sửa lỗi (Fix Bugs)

Chúng ta sẽ thực hiện **2 lần chạy** để thấy rõ sự kỳ diệu của Transaction.

#### Lần 1: Chạy với mã cố tình gây lỗi

1. Restart lại Tomcat Server.
2. Mở trình duyệt và truy cập:  
   `http://localhost:8080/user-management/users?action=test-use-tran`
3. Bạn sẽ thấy lỗi Exception xuất hiện trong Console của IDE.
4. Kiểm tra MySQL (`SELECT * FROM Employee;`):  
   **Kết quả:** Bảng trống trơn (Không có bản ghi nào được chèn vào). Trạng thái ban đầu đã được khôi phục hoàn hảo! Dù 2 lệnh Insert đã chạy qua, nhưng do lệnh Update phía sau bị lỗi nên không có lệnh `commit()` nào được gọi. Rác dữ liệu đã được ngăn chặn thành công.

#### Lần 2: Sửa lỗi để ứng dụng hoạt động chuẩn xác

Mở lại file `UserDAO.java`, tìm đến đoạn gán tham số cho lệnh Update và sửa lại index từ 2 thành 1:

```java
// Đóng (comment) dòng code gây lỗi:
// psUpdate.setBigDecimal(2, new BigDecimal(999.99));

// Sửa lại thành dòng đúng:
psUpdate.setBigDecimal(1, new BigDecimal(999.99));
```

1. Restart lại Tomcat để cập nhật code.
2. Truy cập lại URL trên trình duyệt:  
   `http://localhost:8080/user-management/users?action=test-use-tran`
3. Kiểm tra MySQL (`SELECT * FROM Employee;`):  
   **Kết quả:** Sẽ có 2 bản ghi được chèn vào bảng, và bản ghi của "Quynh" đã được cập nhật thành công mức lương lên 999.99. Tất cả các thao tác đã hội tụ tại một điểm `commit()` duy nhất.

---

## Prompt ra lệnh cho AI Agent triển khai dự án

**Chuẩn bị trước:** Hãy đảm bảo bạn đã biết chính xác đường dẫn thư mục cài đặt Tomcat trên máy tính (Ví dụ: `C:\Tomcat 10.1` trên Windows hoặc `/Library/Tomcat` trên macOS).

Copy đoạn lệnh dưới đây và dán vào khung chat của AI Agent:

```
Đóng vai là một chuyên gia DevOps, hãy giúp tôi tự động hóa quá trình đóng gói và triển khai dự án Java Web hiện tại lên máy chủ Tomcat.

Yêu cầu thực thi:

1. Biên dịch dự án: Mở terminal tại thư mục gốc của dự án này và chạy lệnh mvn clean package để dọn dẹp và đóng gói dự án. Hãy chờ cho đến khi xuất hiện thông báo BUILD SUCCESS.

2. Xác định file: Tìm file .war vừa được tạo ra bên trong thư mục target/ của dự án.

3. Triển khai (Deploy): Sử dụng lệnh hệ thống (terminal/bash/powershell) để copy file .war này và dán vào thư mục webapps của máy chủ Tomcat tại đường dẫn tuyệt đối sau: [ĐIỀN_ĐƯỜNG_DẪN_THƯ_MỤC_TOMCAT_CỦA_BẠN_VÀO_ĐÂY]

4. Khởi động Server: Sau khi copy thành công, hãy điều hướng terminal tới thư mục bin của Tomcat và chạy file khởi động (startup.bat cho Windows hoặc ./startup.sh cho macOS/Linux).

Hãy hiển thị cho tôi các lệnh bạn sẽ chạy trước khi thực thi để tôi phê duyệt.
```
