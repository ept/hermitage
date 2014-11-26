Testing FoundationDB SQL Layer transaction isolation levels
===========================================================

These tests were run with FoundationDB Key-Value Store 2.0.5 and SQL Layer 2.0.2.

Key-Value Store does not use locking to meet its isolation guarantees. Instead, the database uses optimistic concurrency control. It checks for conflicts at commit-time, rejecting transactions that conflict. Typically applications simply try the transaction again. Key-Value Store does not have configurable isolation levels - it is always serializable.

Key-Value Store enforces a 5 second limit on transactions to maintain a reasonable window of time for detecting conflicts. Because of this time constraint, testing isolation manually, like the other tests in Hermitage, is unworkable. So this test utilizes tmux to automate the Hermitage tests.

````bat
#!/bin/bash

if [ -z $1 ]
then
  echo "Specify a test to continue. Options are: g0, g1a, g1b, g1c, otv, pmp, pmp-write, p4, g-single, g-single-dependencies, g-single-write-1, g-single-write-2, g2-item, g2, g2-two-edges"
  exit 2
elif [ -n $1 ]
then
  test=$1
fi

tmux kill-session -t SQL || true
tmux new-session -d -n SQL -s SQL "fdbsqlcli test | tee /tmp/SQL0.out"
tmux split-window -h -t SQL "fdbsqlcli test | tee /tmp/SQL1.out"

count_prompts()
{
  grep "=>" /tmp/SQL$1.out | wc -l
}

wait_for_prompts()
{
  until [ $2 -ne $(count_prompts $1) ]; do
    echo -n '.'
  done
}

tell()
{
  BEFORE=$(count_prompts $1)
  tmux send-keys -t $1 "$2\\;" c-M
  wait_for_prompts $1 $BEFORE
}

wait_for_prompts 0 0
wait_for_prompts 1 0

tell 0 "drop table if exists test"
tell 0 "create table test (id int primary key, value int)"
tell 0 "insert into test (id, value) values (1, 10), (2, 20)"

case $test in
  "g0")
    echo "Running g0 test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "update test set value = 11 where id = 1"
    tell 1 "update test set value = 12 where id = 1"
    tell 0 "update test set value = 21 where id = 2"
    tell 0 "commit"
    tell 0 "select * from test"
    tell 1 "update test set value = 22 where id = 2"
    #Rejected due to optimistic concurrency controls 
    tell 1 "commit"
    tell 0 "select * from test"
    tell 1 "select * from test"
    ;;
  "g1a")
    echo "Running g1a test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "update test set value = 101 where id = 1"
    #Still shows 1 => 10
    tell 1 "select * from test"
    tell 0 "rollback"
    #Still shows 1 => 10
    tell 1 "select * from test"
    tell 1 "commit"
    ;;
  "g1b")
    echo "Running g1b test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "update test set value = 101 where id = 1"
    #Still shows 1 => 10
    tell 1 "select * from test"
    tell 0 "update test set value = 11 where id = 1"
    tell 0 "commit"
    #Still shows 1 => 10. A new transaction must be started to see the commits of other transactions.
    tell 1 "select * from test"
    tell 1 "commit"
    ;;
  "g1c")
    echo "Running g1c test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "update test set value = 11 where id = 1"
    tell 1 "update test set value = 22 where id = 2"
    #Still shows 2 => 20
    tell 0 "select * from test"
    #Still shows 1 => 10
    tell 1 "select * from test"
    tell 0 "commit"
    tell 1 "commit"
    ;;
  "otv")
    echo "Running otv test."
    tmux split-window -h -t SQL "fdbsqlcli test | tee /tmp/SQL2.out"
    wait_for_prompts 2 0
    tell 0 "begin"
    tell 1 "begin"
    tell 2 "begin"
    tell 0 "update test set value = 11 where id = 1"
    tell 0 "update test set value = 19 where id = 2"
    tell 1 "update test set value = 12 where id = 1"
    tell 0 "commit"
    #Shows 1 => 11, 2 => 19. This is because we don't get a snapshot of the db until our first read.
    tell 2 "select * from test"
    tell 1 "update test set value = 18 where id = 2"
    #Still shows 1 => 11, 2 => 19. We're isolated from other transactions now that we have our snapshot.
    tell 2 "select * from test"
    #Rejected due to optimistic concurrency controls
    tell 1 "commit"
    tell 2 "commit"    
    ;;
  "pmp")
    echo "Running pmp test."
    tell 0 "begin"
    tell 1 "begin"
    #Returns nothing
    tell 0 "select * from test where value = 30"
    tell 1 "insert into test (id, value) values(3, 30)"
    tell 1 "commit"
    #Still returns nothing
    tell 0 "select * from test where value = 30"
    tell 0 "commit"
    ;;
  "pmp-write")
    echo "Running pmp-write test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "update test set value = value + 10"
    #Still shows 1 => 10, 2 => 20
    tell 1 "select * from test"
    tell 1 "delete from test where value = 20"
    tell 0 "commit"
    #Shows 1 => 10
    tell 1 "select * from test"
    tell 1 "commit"
    ;;
  "p4")
    echo "Running p4 test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "select * from test where id = 1"
    tell 1 "select * from test where id = 1"
    tell 0 "update test set value = 11 where id = 1"
    tell 1 "update test set value = 11 where id = 1"
    tell 0 "commit"
    #Rejected due to optimistic concurrency controls 
    tell 1 "commit"
    ;;
  "g-single")
    echo "Running g-single test."
    tell 0 "begin"
    tell 1 "begin"
    #Shows 1 => 10
    tell 0 "select * from test where id = 1"
    tell 1 "select * from test where id = 1"
    tell 1 "select * from test where id = 2"
    tell 1 "update test set value = 12 where id = 1"
    tell 1 "update test set value = 18 where id = 2"
    tell 1 "commit"
    #Still shows 2 => 20
    tell 0 "select * from test where id = 2"
    tell 0 "commit"
    ;;
  "g-single-dependencies")
    echo "Running g-single-dependencies test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "select * from test where value % 5 = 0"
    tell 1 "update test set value = 12 where value = 10"
    tell 1 "commit"
    #Returns nothing
    tell 0 "select * from test where value % 3 = 0"
    tell 0 "commit"
    ;;
  "g-single-write-1")
    echo "Running g-single-write-1 test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "select * from test where id = 1"
    tell 1 "select * from test"
    tell 1 "update test set value = 12 where id = 1"
    tell 1 "update test set value = 18 where id = 2"
    tell 1 "commit"
    #Deletes a row
    tell 0 "delete from test where value = 20"
    #Returns nothing
    tell 0 "select * from test where id = 2"
    tell 0 "commit"
    ;;
  "g-single-write-2")
    echo "Running g-single-write-2 test."
    tell 0 "begin"
    tell 1 "begin"
    #Shows 1 => 10
    tell 0 "select * from test where id = 1"
    tell 1 "select * from test"
    tell 1 "update test set value = 12 where id = 1"
    tell 0 "delete from test where value = 20"
    tell 1 "update test set value = 18 where id = 2"
    tell 0 "rollback"
    #Conflicts are checked at commit-time, and the other transaction has not committed, so this goes through successfully.
    tell 1 "commit"
    ;;
  "g2-item")
    echo "Running g2-item test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "select * from test where id in (1,2)"
    tell 1 "select * from test where id in (1,2)"
    tell 0 "update test set value = 11 where id = 1"
    tell 1 "update test set value = 21 where id = 2"
    tell 0 "commit"
    #Rejected due to optimistic concurrency controls
    tell 1 "commit"
    ;;
  "g2")
    echo "Running g2 test."
    tell 0 "begin"
    tell 1 "begin"
    tell 0 "select * from test where value % 3 = 0"
    tell 1 "select * from test where value % 3 = 0"
    tell 0 "insert into test (id, value) values(3, 30)"
    tell 1 "insert into test (id, value) values(4, 42)"
    tell 0 "commit"
    #Rejected due to optimistic concurrency controls
    tell 1 "commit"
    ;;
  "g2-two-edges")
    echo "Running g2-two-edges test."
    tmux split-window -h -t SQL "fdbsqlcli test | tee /tmp/SQL2.out"
    wait_for_prompts 2 0
    tell 0 "begin"
    tell 1 "begin"
    tell 2 "begin"
    tell 0 "select * from test"
    tell 1 "update test set value = value + 5 where id = 2"
    #Still shows 1 => 10, 2 => 20
    tell 2 "select * from test"
    tell 1 "update test set value = 0 where id = 1"
    #Successful commit
    tell 2 "commit"
    #Successful commit
    tell 0 "commit"
    tell 1 "rollback"
    ;;
  *)
    echo "Test not recognized."
esac

tmux attach-session -t SQL

```