--使用ora11g提供的并发包的写法：
--1) 创建过程serial过程，用来被多个并发线程调用
create or replace
procedure TMP_EMP_FLG_UPD( p_lo_rid in rowid, p_hi_rid in rowid )
is
begin
update utable set employee_no='', employee_flag='9'
where rowid between p_lo_rid and p_hi_rid;
end;
/

--2) 按照rowid将表划分为多个chunk，供线程调用
begin
dbms_parallel_execute.create_task('PROCESS BIG TABLE employee_no UPDATE');
dbms_parallel_execute.create_chunks_by_rowid
( task_name => 'PROCESS BIG TABLE employee_no UPDATE',
table_owner => 'USER',
table_name => 'utable',
by_row => false, --不按行记录数而按block数
chunk_size => 2000 );
end;
/

select *
from (
select chunk_id, status, start_rowid, end_rowid
from dba_parallel_execute_chunks
where task_name = 'PROCESS BIG TABLE employee_no UPDATE'
order by chunk_id
)
where rownum <= 5
/
--3) 发起并发任务，按照第2步对表的划分来分配并运行任务
begin
dbms_parallel_execute.run_task
( task_name => 'PROCESS BIG TABLE employee_no UPDATE',
sql_stmt => 'begin TMP_EMP_FLG_UPD( :start_id, :end_id ); end;',
language_flag => DBMS_SQL.NATIVE,
parallel_level => 4 );
end;
/
--4) 删除并发作业 
begin 
dbms_parallel_execute.drop_task('PROCESS BIG TABLE employee_no UPDATE'); 
end; 
/
