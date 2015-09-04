Testing Sophia transaction isolation levels
===========================================

Sophia is a modern embeddable transactional key-value storage.

[http://sphia.org](http://sphia.org)
[https://github.com/pmwkaa/sophia](https://github.com/pmwkaa/sophia)

Being implemented as a small C-written library, Sophia has Append-Only MVCC engine
designed for fast write and read, small to medium key-values.

Originally Sophia got Snapshot Isolation (SI) (v1.2.3) but recently has been upgraded to own
implementation of Serializable Snapshot Isolation (SSI) (v2.1.1).

Key-Value store does not use locking to meet its isolation guarantees.
Instead, the database uses optimistic concurrency control. It checks for conflicts at commit-time,
rejecting transactions that conflict. Typically applications simply try the transaction again.
Key-Value Store does not have configurable isolation levels - it is always Serializable.

Hermitage tests has been included to Sophia test suite:
[https://github.com/pmwkaa/sophia/tree/master/test/functional/hermitage.test.c](https://github.com/pmwkaa/sophia/tree/master/test/functional/hermitage.test.c)

```C
static void set(void*, uint32_t id, uint32_t value);
static void delete(void*, uint32_t id);
static void get(void*, uint32_t id, int value_to_check);
static void *begin(void);
static void commit(void*, int result);
static void rollback(void*);

static void
hermitage_g0(void)
{
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	set(T1, 1, 11);
	set(T2, 1, 12);
	set(T1, 2, 21);
	commit(T1, 0);
	set(T2, 2, 22);
	commit(T2, 1); /* conflict */
	get(st_r.db, 1, 11);
	get(st_r.db, 2, 21);
}

static void
hermitage_g1a(void)
{
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	set(T1, 1, 101);
	get(T2, 1, 10);
	rollback(T1);
	get(T2, 1, 10);
	commit(T2, 0);
}

static void
hermitage_g1b(void)
{
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	set(T1, 1, 101);
	get(T2, 1, 10);
	set(T1, 1, 11);
	commit(T1, 0);
	get(T2, 1, 10);
	commit(T2, 0); /* T1(1) <- T2(1), but T2 is read only */
}

static void
hermitage_g1c(void)
{
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	set(T1, 1, 11);
	set(T2, 2, 22);
	get(T1, 2, 20);
	get(T2, 1, 10);
	commit(T1, 0);
	commit(T2, 0);
}

static void
hermitage_otv(void)
{
	/* observer transaction vanishes */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	void *T3 = begin();
	set(T1, 1, 11);
	set(T1, 2, 19);
	set(T2, 1, 12);
	commit(T1, 0);
	get(T3, 1, 10); /* snapshot created on begin */
	set(T2, 2, 18);
	get(T3, 2, 20);
	commit(T2, 1);  /* rollback on conflict */
	get(T3, 2, 20); /* transaction not sees other updates */
	get(T3, 1, 10);
	commit(T3, 0);
}

static void
hermitage_pmp(void)
{
	/* predicate-many-preceders */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	/* select * from test where value = 30 */
	void *cursor = sp_cursor(st_r.env);
	void *o = sp_object(st_r.db);
	while ((o = sp_get(cursor, o))) {
		uint32_t key = *(uint32_t*)sp_getstring(o, "key", NULL);
		t( key != 30 );
	}
	sp_destroy(cursor);
	set(T2, 3, 30);
	commit(T2, 0);
	get(T1, 1, 10);
	get(T1, 2, 20);
	get(T1, 3, -1);
	commit(T1, 0);
}

static void
hermitage_pmp_write(void)
{
	/* predicate-many-preceders */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	set(T1, 1, 20);
	set(T1, 2, 30);
	get(T2, 1, 10);
	get(T2, 2, 20);
	delete(T2, 2);
	commit(T1, 0);
	get(T2, 1, 10);
	commit(T2, 1); /* conflict */
	get(st_r.db, 1, 20);
	set(st_r.db, 2, 30);
}

static void
hermitage_p4(void)
{
	/* lost update */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	get(T1, 1, 10);
	get(T2, 1, 10);
	set(T1, 1, 11);
	set(T2, 1, 11);
	commit(T1, 0);
	commit(T2, 1); /* conflict */
}

static void
hermitage_g_single(void)
{
	/* read-skew */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	get(T1, 1, 10);
	get(T2, 1, 10);
	get(T2, 2, 20);
	set(T2, 1, 12);
	set(T2, 2, 18);
	commit(T2, 0);
	get(T1, 2, 20);
	commit(T1, 0);
}

static void
hermitage_g2_item(void)
{
	/* write-skew */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	get(T1, 1, 10);
	get(T1, 2, 20);
	get(T2, 1, 10);
	get(T2, 2, 20);
	set(T1, 1, 11);
	set(T2, 1, 21);
	commit(T1, 0);
	commit(T2, 1); /* conflict */
}

static void
hermitage_g2(void)
{
	/* anti-dependency cycles */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	void *T2 = begin();
	/* select * from test where value % 3 = 0 */
	get(T1, 1, 10);
	get(T1, 2, 20);
	get(T2, 1, 10);
	get(T2, 2, 20);
	set(T1, 3, 30);
	set(T2, 4, 42);
	commit(T1, 0);
	commit(T2, 1); /* conflict */
}

static void
hermitage_g2_two_edges0(void)
{
	/* anti-dependency cycles */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	/* select * from test */
	get(T1, 1, 10);
	get(T1, 2, 20);

	void *T2 = begin();
	set(T2, 2, 25);
	commit(T2, 0);

	void *T3 = begin();
	get(T3, 1, 10);
	get(T3, 2, 25);
	commit(T3, 0);

	set(T1, 1, 0);
	commit(T1, 1);
}

static void
hermitage_g2_two_edges1(void)
{
	/* anti-dependency cycles */
	set(st_r.db, 1, 10);
	set(st_r.db, 2, 20);

	void *T1 = begin();
	/* select * from test */
	get(T1, 1, 10);
	get(T1, 2, 20);

	void *T2 = begin();
	set(T2, 2, 25);
	commit(T2, 0);

	void *T3 = begin();
	get(T3, 1, 10);
	get(T3, 2, 25);
	commit(T3, 0);

	/*set(T1, 1, 0);*/
	commit(T1, 0);
}

static void
set(void *dest, uint32_t id, uint32_t value)
{
	void *o = st_object(id, value);
	t( sp_set(dest, o) == 0 );
}

static void
delete(void *dest, uint32_t id)
{
	void *o = st_object(id, id);
	t( sp_delete(dest, o) == 0 );
}

static void
get(void *dest, uint32_t id, int value_to_check)
{
	void *o = st_object(id, id);
	o = sp_get(dest, o);
	if (o == NULL) {
		t( value_to_check == -1 );
		return;
	}
	st_object_is(o, id, value_to_check);
	sp_destroy(o);
}

static void*
begin(void)
{
	void *T = sp_begin(st_r.env);
	t( T != NULL );
	return T;
}

static void
commit(void *dest, int result)
{
	t( sp_commit(dest) == result );
	st_phase();
}

static void
rollback(void *dest)
{
	t( sp_destroy(dest) == 0 );
	st_phase();
}

stgroup *hermitage_group(void)
{
	stgroup *group = st_group("hermitage");
	st_groupadd(group, st_test("g0", hermitage_g0));
	st_groupadd(group, st_test("g1a", hermitage_g1a));
	st_groupadd(group, st_test("g1b", hermitage_g1b));
	st_groupadd(group, st_test("g1c", hermitage_g1c));
	st_groupadd(group, st_test("otv", hermitage_otv));
	st_groupadd(group, st_test("pmp", hermitage_pmp));
	st_groupadd(group, st_test("pmp-write", hermitage_pmp_write));
	st_groupadd(group, st_test("p4", hermitage_p4));
	st_groupadd(group, st_test("g-single", hermitage_g_single));
	st_groupadd(group, st_test("g2-item", hermitage_g2_item));
	st_groupadd(group, st_test("g2", hermitage_g2));
	st_groupadd(group, st_test("g2_two_edges0", hermitage_g2_two_edges0));
	st_groupadd(group, st_test("g2_two_edges1", hermitage_g2_two_edges1));
	return group;
}
```
