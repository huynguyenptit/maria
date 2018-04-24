## Tổ hợp các chỉ mục

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#a-mini-lesson-in-compound-indexes-composite-indexes) bài học về "tổ hợp các chỉ mục"

Tài liệu này bắt đầu tầm thường và có lẽ nhàm chán, nhưng lại đưa đến nhiều thông tin thú vị hơn, không chừng những điều bạn biết về cách MariaDB và lập chỉ mục MySQL hoạt động.

Điều này cũng được giải thích [EXPLAIN][1] (ở một mức độ nào đó).

Hầu hết điều này cũng áp dụng cho các cơ sở dữ liệu không phải MySQL.

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#the-query-to-discuss)Truy vấn để thảo luận

Câu hỏi đặt ra là "Andrew Johnson là tổng thống của Hoa Kỳ khi nào?".

Bảng 'President' sẽ như sau

+-----+------------+----------------+-----------+
| seq | last_name  | first_name     | term      |
+-----+------------+----------------+-----------+
|   1 | Washington | George         | 1789-1797 |
|   2 | Adams      | John           | 1797-1801 |
...
|   7 | Jackson    | Andrew         | 1829-1837 |
...
|  17 | Johnson    | Andrew         | 1865-1869 |
...
|  36 | Johnson    | Lyndon B.      | 1963-1969 |
...

("Andrew Johnson" đã được chọn cho bài học này vì các bản sao.)

Chỉ số nào sẽ là tốt nhất cho câu hỏi đó? Cụ thể hơn, sẽ tốt nhất cho cái gì

    SELECT  term
        FROM  Presidents
        WHERE  last_name = 'Johnson'
          AND  first_name = 'Andrew';
          
Một vài INDEX để thử...

- không indexes
- INDEX(first_name), INDEX(last_name) (2 index riêng lẻ)
- "Index Merge Intersect"
- INDEX(last_name, first_name) (index "gộp lại")
- INDEX(last_name, first_name, term) (index "bao gồm")
- Variants

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#no-indexes)Không indexes

Vâng, tôi đang giả vờ một chút ở đây. Tôi có một khóa CHÍNH trên `seq`, nhưng điều đó không có lợi thế về truy vấn chúng ta đang nghiên cứu

mysql&gt;  SHOW CREATE TABLE Presidents G
CREATE TABLE `presidents` (
  `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
  `last_name` varchar(30) NOT NULL,
  `first_name` varchar(30) NOT NULL,
  `term` varchar(9) NOT NULL,
  PRIMARY KEY (`seq`)
) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8

mysql&gt;  EXPLAIN  SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
+----+-------------+------------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+------------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
+----+-------------+------------+------+---------------+------+---------+------+------+-------------+

# Or, using the other form of display:  EXPLAIN ... G
           id: 1
  select_type: SIMPLE
        table: Presidents
         type: ALL        &lt;-- Implies table scan
possible_keys: NULL
          key: NULL       &lt;-- Implies that no index is useful, hence table scan
      key_len: NULL
          ref: NULL
         rows: 44         &lt;-- That's about how many rows in the table, so table scan
        Extra: Using where

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#implementation-details)Chi tiết triển khai

Trước tiên, hãy mô tả cách InnoDB lưu trữ và sử dụng các chỉ mục.

- Dữ liệu và khóa CHÍNH được "nhóm" lại với nhau trên BTree.
- Tra cứu BTree khá nhanh và hiệu quả. Đối với một bảng hàng triệu có thể có 3 cấp độ của BTree, và hai cấp cao nhất có thể được lưu trữ. 
- Mỗi chỉ số phụ nằm trong một BTree khác, với khóa CHÍNH ở lá.
- Tìm nạp các mục 'liên tiếp' (theo chỉ mục) từ BTree rất hiệu quả vì chúng được lưu trữ liên tiếp. 
- Để đơn giản, chúng ta có thể đếm từng tra cứu BTree dưới dạng 1 đơn vị công việc và bỏ qua các lần quét cho các mục liên tiếp. Điều này xấp xỉ số lần truy cập đĩa cho một bảng lớn trong một hệ thống bận.

Đối với MyISAM, PRIMARY KEY không được lưu trữ với dữ liệu, vì vậy hãy nghĩ về nó như là một khóa thứ cấp (quá đơn giản).

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#indexfirst_name-indexlast_name)INDEX(first_name), INDEX(last_name)

Người mới, một khi anh ta biết về lập chỉ mục, quyết định lập chỉ mục nhiều cột, mỗi cột một lần. Nhưng... 
MySQL hiếm khi sử dụng nhiều hơn một chỉ mục tại một thời điểm trong một truy vấn. Vì vậy, nó sẽ phân tích các chỉ mục khả thi.

- first_name -- có 2 hàng có thể (một do BTree tìm kiếm, sau đó quét liên tục) 
- last_name -- có 2 hàng có thể Giả sử nó chọn last_name. Dưới đây là các bước để thực hiện SELECT: 1. Sử dụng INDEX (last_name), tìm 2 mục chỉ mục với last_name = 'Johnson'. 2. Lấy khóa chính (được bổ sung hoàn toàn vào mỗi chỉ số phụ trong InnoDB); nhận được (17, 36). 3. Tiếp cận dữ liệu bằng cách sử dụng seq = (17, 36) để lấy các hàng cho Andrew Johnson và Lyndon B. Johnson. 4. Sử dụng phần còn lại của mệnh đề WHERE lọc tất cả trừ hàng mong muốn. 5. Đưa ra câu trả lời (1865-1869).

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#index-merge-intersect)"Index Merge Intersect"

OK, bạn thực sự thông minh và quyết định rằng MySQL nên đủ thông minh để sử dụng cả hai chỉ mục tên để nhận được câu trả lời. Điều này được gọi là "Giao lộ". 1. Sử dụng INDEX (last_name), tìm 2 mục chỉ mục với last_name = 'Johnson'; nhận được (7, 17) 2. Sử dụng INDEX (first_name), tìm 2 mục chỉ mục với first_name = 'Andrew'; nhận được (17, 36) 3. "And" hai danh sách với nhau (7,17) & (17,36) = (17) 4. Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để lấy hàng cho Andrew Johnson. 5. Cung cấp câu trả lời (1865-1869).

           id: 1
  select_type: SIMPLE
        table: Presidents
         type: index_merge
possible_keys: first_name,last_name
          key: first_name,last_name
      key_len: 92,92
          ref: NULL
         rows: 1
        Extra: Using intersect(first_name,last_name); Using where

EXPLAIN không cung cấp thông tin chi tiết về số lượng hàng được thu thập từ mỗi chỉ mục, v.v.

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#indexlast_name-first_name)INDEX(last_name, first_name)

Đây được gọi là chỉ mục "hợp chất" hoặc "tổng hợp" vì nó có nhiều hơn một cột. 1. Tìm hiểu về BTree để chỉ mục có được chính xác hàng chỉ mục cho Johnson + Andrew; get seq = (17). 2. Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để lấy hàng cho Andrew Johnson. 3. Cung cấp câu trả lời (1865-1869). Thế này tốt hơn. Trên thực tế, đây thường là cách "tốt nhất"

    ALTER TABLE Presidents
        (drop old indexes and...)
        ADD INDEX compound(last_name, first_name);

           id: 1
  select_type: SIMPLE
        table: Presidents
         type: ref
possible_keys: compound
          key: compound
      key_len: 184             &lt;-- The length of both fields
          ref: const,const     &lt;-- The WHERE clause gave constants for both
         rows: 1               &lt;-- Goodie!  It homed in on the one row.
        Extra: Using where

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#covering-indexlast_name-first_name-term)"Covering": INDEX(last_name, first_name, term)

Thật sự ngạc nhiên! Chúng tôi thực sự có thể làm tốt hơn. Chỉ mục "Bao gồm" là chỉ mục trong đó _tất cả_ của các trường SELECT được tìm thấy trong chỉ mục. Nó có thêm tiền thưởng không cần phải tiếp cận vào "dữ liệu" để hoàn thành nhiệm vụ. 1. Tìm hiểu về BTree để chỉ mục có được chính xác hàng chỉ mục cho Johnson + Andrew; lấy seq = (17). 2. Cung cấp câu trả lời (1865-1869). BTree "dữ liệu" không được chạm vào; đây là một cải tiến về "hợp chất".

    ... ADD INDEX covering(last_name, first_name, term);

           id: 1
  select_type: SIMPLE
        table: Presidents
         type: ref
possible_keys: covering
          key: covering
      key_len: 184
          ref: const,const
         rows: 1
        Extra: Using where; Using index   &lt;-- Note

Mọi thứ đều tương tự như sử dụng "hợp chất", ngoại trừ việc bổ sung "Sử dụng chỉ mục".

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#variants)Biến thể
- Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề WHERE? Trả lời: Thứ tự của những thứ ANDed không quan trọng.
- Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong INDEX? Trả lời: Nó có thể tạo nên sự khác biệt lớn. Xem thêm sau một phút nữa.
- Điều gì sẽ xảy ra nếu có thêm trường vào cuối? Trả lời: Tác hại tối thiểu; có thể rất nhiều (ví dụ, 'bao gồm')..
- Reduncancy? Đó là, nếu bạn có cả hai thứ này: INDEX (a), INDEX (a, b)? Trả lời: Reduncy chi phí một cái gì đó trên INSERTs; nó ít khi hữu ích cho các SELECT.
- Tiếp đầu ngữ? Tức là, INDEX (last_name (5). First_name (5)) Trả lời: Đừng bận tâm; nó hiếm khi giúp ích. (Chi tiết trong một chủ đề khác.)

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#more-examples)Thêm ví dụ:

    INDEX(last, first)
    ... WHERE last = '...' -- good (even though `first` is unused)
    ... WHERE first = '...' -- index is useless

    INDEX(first, last), INDEX(last, first)
    ... WHERE first = '...' -- 1st index is used
    ... WHERE last = '...' -- 2nd index is used
    ... WHERE first = '...' AND last = '...' -- either could be used equally well

    INDEX(last, first)
    Both of these are handled by that one INDEX:
    ... WHERE last = '...'
    ... WHERE last = '...' AND first = '...'

    INDEX(last), INDEX(last, first)
    In light of the above example, don't bother including INDEX(last).

### [](https://gist.github.com/NgVanYud/835237bb79af9a6d013d7d3047b44c57#postlog)Postlog