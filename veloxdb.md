Testing VeloxDB's transaction isolation
=======================================

These tests were run with VeloxDB 0.4.0.

VeloxDB does not support interactive mode. Therefore, this implementation uses VeloxDB in embedded mode, allowing for precise control over transaction execution. With embedded mode, we can initiate multiple transactions in parallel and control when each commits. The engine used in embedded VeloxDB is the same as in regular VeloxDB, ensuring that all isolation guarantees are consistent.

### Running the tests

To run the VeloxDB Hermitage test, follow these steps:

1. Ensure you have .NET 8 SDK installed.

2. Create a new NUnit project and add the VeloxDB.Embedded NuGet package:

    ```bash
    dotnet new nunit
    dotnet add package veloxdb.embedded
    ```

3. Copy the provided test code into `UnitTest1.cs` within the newly created project.

4. Run the test using the following command:

    ```bash
    dotnet test
    ```

This will execute the VeloxDB Hermitage test. All 14 tests should pass.


```cs

using VeloxDB;
using VeloxDB.Embedded;
using VeloxDB.ObjectInterface;

namespace Hermitage;

public class Tests
{
	private VeloxDBEmbedded db;

	[SetUp]
	public void Setup()
	{
		db = new VeloxDBEmbedded("./data", [typeof(VlxTest).Assembly], false);
		using (VeloxDBTransaction trans = db.BeginTransaction())
		{
			VlxTest test1 = trans.ObjectModel.CreateObject<VlxTest>();
			test1.ObjId = 1;
			test1.Value = 10;

			VlxTest test2 = trans.ObjectModel.CreateObject<VlxTest>();
			test2.ObjId = 2;
			test2.Value = 20;

			trans.Commit();
		}
	}

	[TearDown]
	public void Teardown()
	{
		db.Dispose();
		Directory.Delete("./data/system", true);
		Directory.Delete("./data/user", true);
		Directory.Delete("./data");
	}

	[Test]
	public void G0Test()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();
		// T1 updates id=1
		Update(t1, 1, 11);

		// T2 tries to update id=1
		Update(t2, 1, 12);

		// T1 updates id=2
		Update(t1, 2, 21);

		// Serializable prevents T1 from commiting, because it's in conflict with T2
		AssertConflict(() => t1.Commit());

		// T2 now updates id=2
		Update(t2, 2, 22);

		// Commit T2
		t2.Commit();

		// Assert the final state of the records
		AssertDBState(db, [(1, 12), (2, 22)]);
	}

	[Test]
	public void G1aTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		// T1 updates id=1
		Update(t1, 1, 101);

		// T2 reads id=1, should still see the original value (10)
		Assert.That(Read(t2, 1), Is.EqualTo(10));

		// T1 aborts
		t1.Rollback();

		// T2 reads id=1 again, should still see the original value (10)
		Assert.That(Read(t2, 1), Is.EqualTo(10));

		// Commit T2
		t2.Commit();
	}

	[Test]
	public void G1bTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		// T1 updates id=1
		Update(t1, 1, 101);

		// T2 reads id=1 (should still show 10)
		Assert.That(Read(t2, 1), Is.EqualTo(10));

		// T1 updates id=1 again
		Update(t1, 1, 11);

		// Serilizable isolation prevents T1 from committing, it's in conflict with T2
		AssertConflict(() => t1.Commit());

		// T2 reads id=1, it's still 10 since T1 failed
		Assert.That(Read(t2, 1), Is.EqualTo(10));

		// Commit T2
		t2.Commit();

		// Assert final state of the records
		AssertDBState(db, [(1, 10), (2, 20)]);
	}

	[Test]
	public void G1cTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		// T1 updates id=1
		Update(t1, 1, 11);

		// T2 updates id=2
		Update(t2, 2, 22);

		// T1 tries to read id=2 (should still see the old value)
		Assert.That(Read(t1, 2), Is.EqualTo(20));

		// T2 tries to read id=1 (should still see the old value)
		Assert.That(Read(t2, 1), Is.EqualTo(10));

		// Attempt to commit T1, should fail because of serializable isolation
		AssertConflict(() => t1.Commit());

		// Commit T2
		t2.Commit();

		// Assert the final state of the records
		AssertDBState(db, [(1, 10), (2, 22)]);
	}

	[Test]
	public void OTVTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();
		using VeloxDBTransaction t3 = db.BeginTransaction();

		Update(t1, 1, 11);
		Update(t1, 2, 19);
		Update(t2, 1, 12);

		// T3 opened snapshot when transaction started, it is not affected by T1 and T2 changes
		AssertDBState(t3, [(1, 10), (2, 20)]);

		// Serializable prevents commit, conflict with T2 (both changed id=1)
		AssertConflict(() => t1.Commit());

		// T3 still sees the original snapshot
		AssertDBState(t3, [(1, 10), (2, 20)]);

		// T2 is in conflict with T3 (T2 modified data that T3 read)
		AssertConflict(() => t2.Commit());

		// T3 successfully commits, because there are no other transactions in conflict with it.
		t3.Commit();

		// No changes have been made to the database
		AssertDBState(db, [(1, 10), (2, 20)]);
	}

	[Test]
	public void PMPTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		// Ensure there are no objects with Value = 30 in the database at the start of T1
		Assert.That(t1.ObjectModel.GetAllObjects<VlxTest>().Where(obj => obj.Value == 30).Count(), Is.EqualTo(0));

		// T2 inserts a new object with Value = 30, which conflicts with the query from T1
		Insert(t2, 3, 30);

		// Commit should fail due to conflict between T2 and T1
		AssertConflict(()=>t2.Commit());

		// Verify that T1 still sees 0 objects with Value = 30, as T2's commit failed
		Assert.That(t1.ObjectModel.GetAllObjects<VlxTest>().Where(obj => obj.Value == 30).Count(), Is.EqualTo(0));
	}

	[Test]
	public void PMPWriteTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

    	// T1 increments the value of all objects in the database by 10
		foreach(VlxTest obj in t1.ObjectModel.GetAllObjects<VlxTest>())
		{
			obj.Value += 10;
		}

		 // Verify that T2 does not observe the changes made by T1
		AssertDBState(t2, [(1, 10), (2, 20)]);


    	// T2 deletes all objects where the value is 20
		foreach(VlxTest obj in t2.ObjectModel.GetAllObjects<VlxTest>().Where(obj=>obj.Value == 20))
		{
			obj.Delete();
		}

    	// T1 fails to commit due to conflict with T2
		AssertConflict(()=>t1.Commit());

    	// Verify that T2 observes its own changes and no other updates
		AssertDBState(t2, [(1, 10)]);
		t2.Commit();

		// Verify that after T2 commits, its changes are globally visible
		AssertDBState(db, [(1, 10)]);
	}

	[Test]
	public void P4Test()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		Read(t1, 1);
		Read(t2, 1);

		Update(t1, 1, 11);
		Update(t2, 1, 11);

    	// T1 fails to commit due to conflict with T2
		AssertConflict(() => t1.Commit());

    	// T2 successfully commits
		t2.Commit();
		AssertDBState(db, [(1, 11), (2, 20)]);
	}

	[Test]
	public void GSingleTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		Assert.That(Read(t1, 1), Is.EqualTo(10));
		Assert.That(Read(t2, 1), Is.EqualTo(10));
		Assert.That(Read(t2, 2), Is.EqualTo(20));

		Update(t2, 1, 12);
		Update(t2, 2, 18);

		// T2 fails to commit due to conflict with T1
		AssertConflict(()=>t2.Commit());

		Assert.That(Read(t1, 2), Is.EqualTo(20));

		// T1 successfully commits
		t1.Commit();
	}

	[Test]
	public void GSingleDependenciesTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

    	// T1 retrieves all objects where the value is divisible by 5
		VlxTest[] result = t1.ObjectModel.GetAllObjects<VlxTest>().Where(obj=>obj.Value % 5 == 0).ToArray();

    	// T2 modifies objects where the value is 10
		foreach (VlxTest obj in t2.ObjectModel.GetAllObjects<VlxTest>().Where(obj=>obj.Value == 10))
		{
			obj.Value = 12;
		}

		// T2 fails to commit, due to conflict with T1
		AssertConflict(() => t2.Commit());

		Assert.That(t1.ObjectModel.GetAllObjects<VlxTest>().Where(obj => obj.Value % 3 == 0).Count(), Is.EqualTo(0));
	}

	[Test]
	public void GSingleWriteTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		Assert.That(Read(t1, 1), Is.EqualTo(10));

		AssertDBState(t2, [(1, 10), (2, 20)]);
		Update(t2, 1, 12);
		Update(t2, 2, 18);

		// T2 fails to commit, due to conflict with T1
		AssertConflict(()=>t2.Commit());

		foreach (VlxTest obj in t1.ObjectModel.GetAllObjects<VlxTest>().Where(obj=>obj.Value == 20))
		{
			obj.Delete();
		}

		Assert.That(Exists(t1, 2), Is.False);

		// Successfully commits
		t1.Commit();

		AssertDBState(db, [(1, 10)]);
	}

	[Test]
	public void G2ItemTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

		t1.ObjectModel.GetAllObjects<VlxTest>().Where(obj => obj.Value % 3 == 0).ToArray();
		t2.ObjectModel.GetAllObjects<VlxTest>().Where(obj => obj.Value % 3 == 0).ToArray();

		Update(t1, 1, 11);
		Update(t2, 2, 21);

		// T1 fails to commit, due to conflict with T2
		AssertConflict(()=>t1.Commit());

		// T2 successfully commits
		t2.Commit();
	}

	[Test]
	public void G2Test()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();
		using VeloxDBTransaction t2 = db.BeginTransaction();

    	// Both T1 and T2 retrieve all objects where the value is divisible by 3
		t1.ObjectModel.GetAllObjects<VlxTest>().Where(obj => obj.Value % 3 == 0).ToArray();
		t2.ObjectModel.GetAllObjects<VlxTest>().Where(obj => obj.Value % 3 == 0).ToArray();

		Insert(t1, 3, 30);
		Insert(t2, 4, 42);

		// T1 fails to commit due to conflict with T2
		AssertConflict(()=>t1.Commit());

		// T2 successfully commits
		t2.Commit();

		AssertDBState(db, [(1, 10), (2, 20), (4, 42)]);
	}

	[Test]
	public void G2TwoEdgesTest()
	{
		using VeloxDBTransaction t1 = db.BeginTransaction();

		// Retrieve all objects in T1
		t1.ObjectModel.GetAllObjects<VlxTest>().ToArray();

		using VeloxDBTransaction t2 = db.BeginTransaction();
		Update(t2, 2, obj => obj.Value += 5);

		// T2 Fails to commit, due to conflict with T1
		AssertConflict(()=>t2.Commit());

		using VeloxDBTransaction t3 = db.BeginTransaction();

		// Since T2 was rolled back and T1 is not yet committed, T3 sees the initial state
		AssertDBState(t3, [(1, 10), (2, 20)]);
		t3.Commit();

		Update(t1, 1, 0);

		// T3 has committed and has not modified any data T1 depends on. T1 can commit, and the total order of transactions is T3, T1
		t1.Commit();
	}

	static void Update(VeloxDBTransaction t, int objId, Action<VlxTest> update)
	{
		HashIndexReader<VlxTest, int> index = t.ObjectModel.GetHashIndex<VlxTest, int>("ObjId");
		update(index.GetObject(objId));
	}


	static void Update(VeloxDBTransaction t, int objId, int value)
	{
		HashIndexReader<VlxTest, int> index = t.ObjectModel.GetHashIndex<VlxTest, int>("ObjId");
		index.GetObject(objId).Value = value;
	}

	static int Read(VeloxDBTransaction t, int objId)
	{
		HashIndexReader<VlxTest, int> index = t.ObjectModel.GetHashIndex<VlxTest, int>("ObjId");
		return index.GetObject(objId).Value;
	}

	static bool Exists(VeloxDBTransaction t, int id)
	{
		HashIndexReader<VlxTest, int> index = t.ObjectModel.GetHashIndex<VlxTest, int>("ObjId");
		return index.GetObject(id) != null;
	}

	static void Insert(VeloxDBTransaction t, int objId, int value)
	{
		VlxTest newObj = t.ObjectModel.CreateObject<VlxTest>();
		newObj.ObjId = objId;
		newObj.Value = value;
	}

	private void AssertDBState(VeloxDBTransaction t, (int id, int value)[] expected)
	{
		Dictionary<int, int> existing = t.ObjectModel.GetAllObjects<VlxTest>().ToDictionary(x => x.ObjId, x => x.Value);

		Assert.That(existing.Count, Is.EqualTo(expected.Length));

		foreach (var tuple in expected)
		{
			Assert.That(existing.ContainsKey(tuple.id), Is.True);
			Assert.That(existing[tuple.id], Is.EqualTo(tuple.value));
		}
	}


	private void AssertDBState(VeloxDBEmbedded db, (int id, int value)[] expected)
	{
		using VeloxDBTransaction readTrans = db.BeginTransaction(TransactionType.Read);
		AssertDBState(readTrans, expected);
	}

	private static void AssertConflict(TestDelegate testDelegate)
	{
		DatabaseException exception = Assert.Throws<DatabaseException>(testDelegate);
		Assert.That(exception.Detail.ErrorType, Is.EqualTo(DatabaseErrorType.Conflict));
	}
}

[DatabaseClass]
[HashIndex("ObjId", true, nameof(VlxTest.ObjId))]
public abstract class VlxTest : DatabaseObject
{
	[DatabaseProperty]
	public abstract int ObjId { get; set; }

	[DatabaseProperty]
	public abstract int Value { get; set; }
}

```