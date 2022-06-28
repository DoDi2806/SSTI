# Server-Side Template Injection

# I****. Server-side template injection là gì?****

Việc nhúng các đầu vào từ phía người dùng theo cách không an toàn vào trong templates dẫn đến Server-Side Template Injection - một lỗ hổng nghiêm trọng thường xuyên dễ dàng bị nhầm lẫn với Cross-Site Scripting (XSS), hoặc hoàn toàn bị ngó lơ.

Không giống như XSS, Template Injection có thể được sử dụng để tấn công trực tiếp vào bên trong máy chủ web và thường bao gồm Remote Code Execution (RCE) - thực thi mã từ xa.

# II****. Server-side template injection xảy ra khi nào?****

Server-side template injection xảy ra khi những nội dung được nhập vào từ phía người dùng được nhúng không an toàn vào template ở phía máy chủ, cho phép người sử dụng có thể inject template trực tiếp. Bằng cách sử dụng các template độc hại , kẻ tấn công có thể thực thi mã tùy ý và kiểm soát hoàn toàn web server.

# III****. Quy trình của một cuộc tấn công****

## 1. Detect

### 1.1. Plaintext context

• Lỗi thường sẽ xuất hiện theo một trong những phương thức sau:

***smarty=Hello {[user.name](http://user.name/)}
Hello user1***

***freemarker=Hello ${username}
Hello newuser***

• Chúng ta có thể gửi các payload sử dụng các toán tử cơ bản để phát hiện nhiều template engine bằng 1 câu HTTP request duy nhất.

***smarty=Hello ${7*7}
Hello 49***

***freemarker=Hello ${7*7}
Hello 49***

### 1.2. Code context

• Dữ liệu đầu vào của người dùng cũng có thể được đặt trong một template statement, thường là tên biến.

***personal_greeting=username
Hello user01***

• Biến này thậm chí còn dễ bỏ sót hơn trong quá trình đánh giá, vì nó không dẫn đến XSS một cách rõ ràng. Thay đổi giá trị của username thường sẽ trả về kết quả rỗng hoặc gây lỗi ứng dụng. Trường hợp này có thể được phát hiện bằng cách xác minh các tham số không thể XSS trực tiếp, sau đó thoát ra khỏi template statement và thêm thẻ HTML vào sau nó:

***personal_greeting=username<tag>
Hello***

***personal_greeting=username}}<tag>
Hello user01 <tag>***

## 2. ****Identify****

• Sau khi phát hiện được template injection , bước tiếp theo là xác định template engine đang được sử dụng. Mũi tên xanh và đỏ tương ứng lần lượt với response 'success' và 'failure'. Trong một số trường hợp, một payload có thể có nhiều response thành công khác nhau.

• Ví dụ: test với input {{7 * '7'}} sẽ dẫn đến:

49 trong Twig.

7777777 trong Jinja2.

Không trả về gì nếu không có template engine nào đước sử dụng.

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled.png)

## 3. ****Exploit****

### 1. Read

Sau khi xác định được template engine, việc tiếp theo đọc tài liệu liên quan. Các nội dung cần quan tâm chính là:

- Cú pháp cơ bản của template engine.
- Danh sách các phương thức, hàm, bộ lọc và biến đã được dựng sẵn.
- Danh sách các tiện ích mở rộng / plugin.

### 2. Explore

Bước này, việc chúng ta cần làm là tìm ra chính xác những gì có thể truy cập được.

- Xem xét cả những objects mặc định được cung cấp bởi template engine lẫn các objects dành riêng cho ứng dụng được truyền vào template bởi các nhà phát triển.
- Bruteforce các tên biến. Các object do nhà phát triển cung cấp có khả năng chứa các thông tin nhạy cảm.

### 3. Attack

Xem xét từng function để tìm các lỗ hổng có thể khai thác. Các dạng tấn công có thể là tạo object tùy ý, đọc / ghi file tùy ý (bao gồm cả remote file), hay khai thác lỗ hổng leo thang đặc quyền.

# IV. Demo

• Thấy có text-box thì check XSS đơn giản thì không có kết quả.

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled%201.png)

• Nhìn vào đây có thể thấy hàm này sử dụng ajax trong jquery để gửi data từ client lên server.

• Nếu request thành công thì nó sẽ trả lại dữ liệu trong "data" và đổ vào thẻ <div> có id là "result", ngược lại thì đưa ra thông báo lỗi "An error occurs!".

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled%202.png)

• Thử 1 payload đơn giản thì ứng dụng trả về “6”, xảy ra lỗi SSTI.

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled%203.png)

• Báo error thì có thể đoán ứng dụng sử dụng engine FreeMarker hoặc Smarty. Twig có dạng ${{...}}

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled%204.png)

• Search Google tìm thấy 1 payload của FreeMarker có thể thực thi mã từ xa.

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled%205.png)

• Kết quả trả về sau khi inject payload trên.

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled%206.png)

• <#assign> cho phép định nghĩa biến ngay trong template. Đoạn code trên tạo một tên biến là "ex", việc sử dụng Built-in `"freemarker.template.utility.Execute"?new()` cho phép tạo một object tùy ý, chính là object của "Excute" Class được implement từ "TemplateModel".

• Bắt request trong Burp, thực thi mã từ xa thành công.

![Untitled](Server-Side%20Template%20Injection%2065c90dacbc0e453383387b9ff3fd2bfa/Untitled%207.png)