### 1. Enable mail sender.

#####  Bạn cần định nghĩa IP của goldwing mail trong host file:
- Linux (dưới quyền root) `/etc/hosts`
```
vi /etc/hosts
```

- Windows `c:\Windows\System32\drivers\etc\hosts` dưới quyền admin

Thêm IP/Hostname như sau:
```
192.168.xxx.xxx  goldwing.shinhan.com
```

##### Kết nối tới OS bằng user oracle
- Linux sử dụng SSH tool
```
sqlplus / as sysdba
```

##### Chuyển session tới SID/Service_name cần check datafile (Ví dụ SID/ServiceName là `PAMDB`):
```
SQL> alter session set container=PAMDB;
```

##### Chạy các câu lệnh sau để create các package mail.

```
SQL> 
@$ORACLE_HOME/rdbms/admin/utlmail.sql
@$ORACLE_HOME/rdbms/admin/prvtmail.plb
ALTER SYSTEM SET smtp_out_server='goldwing.shinhan.com' SCOPE=both;
```

--------------------------------
### 2. Bạn cần tạo một stored procedure để kiểm tra số lượng datafile trong tablespace USERS và tính toán tổng dung lượng đã sử dụng.

##### Tạo procedure send mail report kết quả khi nào tự động tạo thêm datafile.
```sql
CREATE OR REPLACE PROCEDURE send_mail (p_recipients  IN VARCHAR2, 
                                       p_subject  IN VARCHAR2, 
                                       p_message   IN VARCHAR2)
AS
BEGIN  
    UTL_MAIL.send(
    sender     => 'admin@goldwing.shinhan.com',
    recipients => p_recipients,
    --cc         => 'person3@domain.com',
    --bcc        => 'myboss@domain.com',
    subject    =>  '[FASTDB] ' || p_subject,
    message    => p_message );
END;
```
##### Tạo procedure `add_datafile_to_ts` tính tự động tính toán:
>  lưu ý thay đổi các tham số của store trước khi compile:
*   tên các tablespace của bạn trong tham số này: `v_tablespace_list` ví dụ bên dưới theo dõi các table space cụ thể như sau: `'USERS', 'PAPERLESS', 'TS_OCR'`
*  `v_percent_max` : Là cho phép tham số % tối đa tổng dung lượng của datafile
*  `v_mail_recipients`: danh sách mail nhận thông báo

```sql
CREATE OR REPLACE PROCEDURE add_datafile_to_ts AS
  v_tablespace_list SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST('USERS', 'PAPERLESS', 'TS_OCR');
  v_max_size NUMBER := 32; -- Base on block size of DB: 8k = 8*2^22 = 32GB in bytes
  v_total_size NUMBER := 0;
  v_total_files NUMBER := 0;
  v_percent_used NUMBER := 0;
  v_percent_max NUMBER := 95;
  v_mail_recipients varchar2(500);
  v_mail_subject varchar2(200);
  v_mail_message varchar2(500);
BEGIN
  v_mail_recipients := 'sh18506320@goldwing.shinhan.com; sh10504877@goldwing.shinhan.com; sh22506251@goldwing.shinhan.com';
  v_mail_subject    := TO_CHAR(SYSDATE,'YYYYMMDD') || ' Report of add new datafile of service ' || SYS_CONTEXT('USERENV','SERVICE_NAME');
  v_mail_message    := 'Auto add a new datafile information if percent used is greater than '|| v_percent_max || '%.' || CHR(10) || 'Information:';
  
  FOR i IN 1..v_tablespace_list.COUNT LOOP
    -- get total size and number of datafiles in tablespace
    SELECT SUM(bytes)/1024/1024/1024, COUNT(*)
    INTO v_total_size, v_total_files
    FROM dba_data_files
    WHERE tablespace_name = v_tablespace_list(i);

    -- calculate percent used
    v_percent_used := round((v_total_size/(v_total_files * v_max_size))*100);

    -- if percent used is greater than v_percent_max, add a new datafile
    IF v_percent_used >= v_percent_max THEN
      EXECUTE IMMEDIATE 'ALTER TABLESPACE ' || v_tablespace_list(i) ||
        ' ADD DATAFILE SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED';
        -- Send mail report to related
        v_mail_message :=  v_mail_message 
                        || CHR(10) || ' SERVICE_NAME: ' || SYS_CONTEXT('USERENV','SERVICE_NAME')
                        || CHR(10) || ' Tablespace: ' || v_tablespace_list(i) 
                        || CHR(10) || ' Percent_used: ' || v_percent_used 
                        || CHR(10) || ' Total_files: ' || v_total_files 
                        || CHR(10) || ' Datetime: ' || TO_CHAR(SYSDATE,'YYYY-MM-DD HH24:MI:SS');
        send_mail(p_recipients  => v_mail_recipients,
                  p_subject     => v_mail_subject,
                  p_message     => v_mail_message);
    END IF;
  END LOOP;
END;
/
/
```

###### Lưu ý rằng để sử dụng sys.odcivarchar2list, bạn cần phải có quyền truy cập vào package SYS.ODCIVARCHAR2LIST.
Bạn có thể kiểm tra quyền truy cập của người dùng vào package SYS.ODCIVARCHAR2LIST bằng cách sử dụng câu lệnh SQL sau:

```sql
SELECT * FROM dba_tab_privs WHERE table_name = 'ODCIVARCHAR2LIST' AND owner = 'SYS';
```

Câu lệnh này sẽ trả về tất cả các quyền được cấp cho người dùng trên package SYS.ODCIVARCHAR2LIST. Nếu kết quả trả về không có bất kỳ dòng nào, có thể người dùng không có quyền truy cập vào package này.

Nếu người dùng không có quyền truy cập vào package SYS.ODCIVARCHAR2LIST, quản trị viên có thể cấp quyền cho người dùng bằng cách chạy câu lệnh sau:

```sql
GRANT EXECUTE ON SYS.ODCIVARCHAR2LIST TO <username>;
```


###### Để gọi stored procedure test này với danh sách tablespace cần kiểm tra, bạn có thể sử dụng câu lệnh SQL như sau:
```sql
BEGIN
  add_datafile_to_ts();
END;
/
```

##### Để tạo một job sử dụng Oracle Scheduler để thực thi stored procedure add_datafile_to_ts định kỳ mỗi 10 phút, bạn có thể sử dụng câu lệnh SQL sau:

```sql
BEGIN
  -- create a new job
  DBMS_SCHEDULER.CREATE_JOB (
    job_name => 'ADD_DATAFILE_TO_TS_JOB',
    job_type => 'STORED_PROCEDURE',
    job_action => 'add_datafile_to_ts',
    start_date => SYSTIMESTAMP,
    repeat_interval => 'FREQ=MINUTELY;INTERVAL=10',
    enabled => TRUE,
    comments => 'Automatically add datafiles to tablespaces when usage exceeds 90%'
  );
END;
/
```

##### Trong câu lệnh này, repeat_interval được thiết lập để chạy job mỗi 10 phút bằng cách sử dụng FREQ=MINUTELY và INTERVAL=10. Bạn cũng có thể sử dụng các giá trị khác cho FREQ và INTERVAL, ví dụ như FREQ=HOURLY và INTERVAL=1 để chạy job mỗi giờ.

###### Lưu ý rằng để sử dụng Oracle Scheduler, bạn cần phải kích hoạt nó trên cơ sở dữ liệu của mình bằng cách chạy các lệnh sau:

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_SCHEDULER_JOB (
    job_name => 'ENABLE_SCHEDULER_JOB',
    job_type => 'PLSQL_BLOCK',
    job_action => 'BEGIN DBMS_SCHEDULER.START_SCHEDULER(); END;',
    start_date => SYSTIMESTAMP,
    enabled => TRUE
  );
END;
/
```
##### Tham khảo câu lệnh
```sql
    SELECT 
    tablespace_name ,
    COUNT(*) ,
    (SUM(bytes)/1024/1024/1024/(COUNT(*) * 32))*100,
    round(SUM(bytes)/1024/1024/1024) as Total_Used_DB,
    round((SUM(bytes)/1024/1024/1024/(COUNT(*) * 32))*100) as Percent_Used,
    (COUNT(*) * 32) as Max_size_DB,
    TO_CHAR(SYSDATE,'YYYY-MM-DD HH24:MI:SS') as Datetime
    --INTO v_total_size, v_total_files
    FROM dba_data_files
    WHERE tablespace_name IN ('USERS', 'PAPERLESS', 'TS_OCR')
    group by tablespace_name;
```
