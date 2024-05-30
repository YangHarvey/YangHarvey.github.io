---
title: Postgres源码解析---index.c
date: 2024-05-08 00:00:00
tags: [postgres, index]
description: Postgres源码解析---index.c
layout: post
---

# Postgres源码解析---index.c
本文将会解析Postgres中的index.h和index.c部分代码，这部分代码的功能是管理PG中的索引，包含创建和销毁索引等等。

## 1. index.c

这部分代码负责用于管理pg中的索引，其中最重要的功能就是创建索引和删除索引，下面我们就来简单介绍一下创建索引index_create函数的流程。index_create会接收一系列的参数，然后做以下几个操作：

- 属性继承：继承自表的一些属性，例如namespace等
- 参数检查：检查参数是否正确
- 创建索引对象并更新系统表：获取索引oid，随后更新系统表包括pg_class, pg_attribute, pg_index等等
- 填充索引：index_build

### 参数

Relation heapRelation：表

 const char *indexRelationName：索引名

 Oid indexRelationId：索引的oid，可以由外部指定

IndexInfo *indexInfo：索引的描述信息

```c
Oid
index_create(Relation heapRelation,
			 const char *indexRelationName,
			 Oid indexRelationId,
			 Oid parentIndexRelid,
			 Oid parentConstraintId,
			 RelFileNumber relFileNumber,
			 IndexInfo *indexInfo,
			 List *indexColNames,
			 Oid accessMethodObjectId,
			 Oid tableSpaceId,
			 Oid *collationObjectId,
			 Oid *classObjectId,
			 int16 *coloptions,
			 Datum reloptions,
			 bits16 flags,
			 bits16 constr_flags,
			 bool allow_system_table_mods,
			 bool is_internal,
			 Oid *constraintId)
```

## 2. 属性继承

这部分代码负责获取索引的一些属性，例如索引的namespace应该和原表的namespace保持一致，其余属性包括shared_relation，mapped_relation等等也和原表保持一致

```c
	/*
	 * The index will be in the same namespace as its parent table, and is
	 * shared across databases if and only if the parent is.  Likewise, it
	 * will use the relfilenumber map if and only if the parent does; and it
	 * inherits the parent's relpersistence.
	 */
	namespaceId = RelationGetNamespace(heapRelation);
	shared_relation = heapRelation->rd_rel->relisshared;
	mapped_relation = RelationIsMapped(heapRelation);
	relpersistence = heapRelation->rd_rel->relpersistence;
```

## 3. 参数检查

参数检查这一步会根据参数判断索引创建是否合法，并拒绝不合理（或者不支持）的索引创建请求。

- 拒绝创建具有非确定性排序规则的索引
- 拒绝在系统表上创建并发索引
- 拒绝在系统表上创建并发索引
- 不支持带排他约束的并发索引
- 拒绝在initdb之后为共享关系创建索引
- 检查索引名是否重复
- 检查约束名是否重复

### 3.1 拒绝创建具有非确定性排序规则的索引

btree text_pattern_ops使用 text_eq 作为等式运算符，只要排序规则是确定性的。text_eq 随后就简化为按位等同，因此在语义上与该操作符类中的其他运算符和函数兼容。但是，如果使用非确定性排序规则，text_eq 可能会产生与索引的实际行为（由操作符类的比较函数决定）不兼容的结果。我们通过拒绝创建具有该操作符类和非确定性排序规则的索引来防止这种问题。

```c
	/*
	 * Btree text_pattern_ops uses text_eq as the equality operator, which is
	 * fine as long as the collation is deterministic; text_eq then reduces to
	 * bitwise equality and so it is semantically compatible with the other
	 * operators and functions in that opclass.  But with a nondeterministic
	 * collation, text_eq could yield results that are incompatible with the
	 * actual behavior of the index (which is determined by the opclass's
	 * comparison function).  We prevent such problems by refusing creation of
	 * an index with that opclass and a nondeterministic collation.
	 *
	 * The same applies to varchar_pattern_ops and bpchar_pattern_ops.  If we
	 * find more cases, we might decide to create a real mechanism for marking
	 * opclasses as incompatible with nondeterminism; but for now, this small
	 * hack suffices.
	 *
	 * Another solution is to use a special operator, not text_eq, as the
	 * equality opclass member; but that is undesirable because it would
	 * prevent index usage in many queries that work fine today.
	 */
	for (i = 0; i < indexInfo->ii_NumIndexKeyAttrs; i++)
	{
		Oid			collation = collationObjectId[i];
		Oid			opclass = classObjectId[i];

		if (collation)
		{
			if ((opclass == TEXT_BTREE_PATTERN_OPS_OID ||
				 opclass == VARCHAR_BTREE_PATTERN_OPS_OID ||
				 opclass == BPCHAR_BTREE_PATTERN_OPS_OID) &&
				!get_collation_isdeterministic(collation))
			{
				HeapTuple	classtup;

				classtup = SearchSysCache1(CLAOID, ObjectIdGetDatum(opclass));
				if (!HeapTupleIsValid(classtup))
					elog(ERROR, "cache lookup failed for operator class %u", opclass);
				ereport(ERROR,
						(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
						 errmsg("nondeterministic collations are not supported for operator class \"%s\"",
								NameStr(((Form_pg_opclass) GETSTRUCT(classtup))->opcname))));
				ReleaseSysCache(classtup);
			}
		}
	}
```

### 3.2 拒绝在系统表上创建并发索引

什么是并发索引？并发索引即在创建索引时不阻塞其他事务对原表的读写

为什么拒绝?因为并发索引创建可能会因为锁的提前释放而导致数据不一致

```c
	/*
	 * Concurrent index build on a system catalog is unsafe because we tend to
	 * release locks before committing in catalogs.
	 */
	if (concurrent &&
		IsCatalogRelation(heapRelation))
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("concurrent index creation on system catalog tables is not supported")));
```

### 3.3 不支持带排他约束的并发索引

什么是排他约束的索引？创建索引有两种情况，第一种情况就是操作者手动创建索引，但是还有第二种情况，就是当表中存在某些约束，比如主键约束,unique约束等等，系统自动为其创建的索引。

```c
	/*
	 * This case is currently not supported.  There's no way to ask for it in
	 * the grammar with CREATE INDEX, but it can happen with REINDEX.
	 */
	if (concurrent && is_exclusion)
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("concurrent index creation for exclusion constraints is not supported")));
```

### 3.4 拒绝在initdb之后为共享关系创建索引

共享关系是跨多个数据库共享的，它们在initdb过程中被创建，并且其元数据需要在所有数据库的pg_class表中一致。在initdb之后创建共享索引会导致不一致。

```c
	/*
	 * We cannot allow indexing a shared relation after initdb (because
	 * there's no way to make the entry in other databases' pg_class).
	 */
	if (shared_relation && !IsBootstrapProcessingMode())
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("shared indexes cannot be created after initdb")));
```

### 3.5 检查索引名是否重复

如果有index_create_if_not_exists标志位，则跳过，否则报错

```c
	/*
	 * Check for duplicate name (both as to the index, and as to the
	 * associated constraint if any).  Such cases would fail on the relevant
	 * catalogs' unique indexes anyway, but we prefer to give a friendlier
	 * error message.
	 */
	if (get_relname_relid(indexRelationName, namespaceId))
	{
		if ((flags & INDEX_CREATE_IF_NOT_EXISTS) != 0)
		{
			ereport(NOTICE,
					(errcode(ERRCODE_DUPLICATE_TABLE),
					 errmsg("relation \"%s\" already exists, skipping",
							indexRelationName)));
			table_close(pg_class, RowExclusiveLock);
			return InvalidOid;
		}

		ereport(ERROR,
				(errcode(ERRCODE_DUPLICATE_TABLE),
				 errmsg("relation \"%s\" already exists",
						indexRelationName)));
	}
```

### 3.6 检查约束名是否重复

```c
	if ((flags & INDEX_CREATE_ADD_CONSTRAINT) != 0 &&
		ConstraintNameIsUsed(CONSTRAINT_RELATION, heapRelationId,
							 indexRelationName))
	{
		/*
		 * INDEX_CREATE_IF_NOT_EXISTS does not apply here, since the
		 * conflicting constraint is not an index.
		 */
		ereport(ERROR,
				(errcode(ERRCODE_DUPLICATE_OBJECT),
				 errmsg("constraint \"%s\" for relation \"%s\" already exists",
						indexRelationName, RelationGetRelationName(heapRelation))));
	}
```

## 4. 创建索引oid并更新系统表：

### 4.1 构建indexTupDesc

为什么需要indexTupDesc？这个其实很好理解，因为数据库需要从原表构建索引的tuple，这个构建过程需要一定的规则，indexTupDesc就是用来描述索引tuple的结构，方面之后的构建过程

```c
	/*
	 * construct tuple descriptor for index tuples
	 */
	indexTupDesc = ConstructTupleDescriptor(heapRelation,
											indexInfo,
											indexColNames,
											accessMethodObjectId,
											collationObjectId,
											classObjectId);
```

### 4.2 为index分配Oid

如果给定的参数Oid不是非法的，那么就是用给定的Oid创建索引，否则就自动分配一个

```c
	/*
	 * Allocate an OID for the index, unless we were told what to use.
	 *
	 * The OID will be the relfilenumber as well, so make sure it doesn't
	 * collide with either pg_class OIDs or existing physical files.
	 */
	if (!OidIsValid(indexRelationId))
	{
		/* Use binary-upgrade override for pg_class.oid and relfilenumber */
		if (IsBinaryUpgrade)
		{
			if (!OidIsValid(binary_upgrade_next_index_pg_class_oid))
				ereport(ERROR,
						(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
						 errmsg("pg_class index OID value not set when in binary upgrade mode")));

			indexRelationId = binary_upgrade_next_index_pg_class_oid;
			binary_upgrade_next_index_pg_class_oid = InvalidOid;
```

### 4.3 创建索引relcache条目，必要时创建diskfile

什么是relcache呢？

对于一个PostgreSQL系统来说，对于系统表和普通表模式的访问是非常频繁的。提高这些访问的效率，PostgreSQL设立了高速缓存(Cache)。Cache中包括一个**系统表元组Cache(SysCache)**和一个**表模式信息Cache(RelCache)**。其中：

- SysCache中存放的是最近使用过的**系统表的元组**；
- RelCache中包含所有最近访问过的**表的模式信息(包含系统表的信息)**。

值得注意的是，两种Cache都不是所有进程共享的。每一个PostgreSQL的进程都维护着自己的SysCache和RelCache。

```c
	/*
	 * create the index relation's relcache entry and, if necessary, the
	 * physical disk file. (If we fail further down, it's the smgr's
	 * responsibility to remove the disk file again, if any.)
	 */
	indexRelation = heap_create(indexRelationName,
								namespaceId,
								tableSpaceId,
								indexRelationId,
								relFileNumber,
								accessMethodObjectId,
								indexTupDesc,
								relkind,
								relpersistence,
								shared_relation,
								mapped_relation,
								allow_system_table_mods,
								&relfrozenxid,
								&relminmxid,
								create_storage);
```

### 4.4 更新pg_class系统表

这里在更新完pg_class表之后就将其关闭，并在上面加行级排他锁，这样在索引构建完成之前，这个索引对其他事务都是不可见的。

```c
	/*
	 * store index's pg_class entry
	 */
	InsertPgClassTuple(pg_class, indexRelation,
					   RelationGetRelid(indexRelation),
					   (Datum) 0,
					   reloptions);

	/* done with pg_class */
	table_close(pg_class, RowExclusiveLock);
```

### 4.5 更新pg_attribute表

```c
	/*
	 * now update the object id's of all the attribute tuple forms in the
	 * index relation's tuple descriptor
	 */
	InitializeAttrbuteOids(indexRelation,
							indexInfo->ii_NumIndexAttrs,
							indexRelationId);

	/*
	 * append ATTRIBUTE tuples for the index
	 */
	AppendAttributeTuples(indexRelation, indexInfo->ii_OpclassOptions);
```

### 4.6 更新pg_index系统表

```c
	/* ----------------
	 *	  update pg_index
	 *	  (append INDEX tuple)
	 *
	 *	  Note that this stows away a representation of "predicate".
	 *	  (Or, could define a rule to maintain the predicate) --Nels, Feb '92
	 * ----------------
	 */
	UpdateIndexRelation(indexRelationId, heapRelationId, parentIndexRelid,
						indexInfo,
						collationObjectId, classObjectId, coloptions,
						isprimary, is_exclusion,
						(constr_flags & INDEX_CONSTR_CREATE_DEFERRABLE) == 0,
						!concurrent && !invalid,
						!concurrent);
```

### 4.7 为新创建的索引注册约束和依赖关系

为索引注册约束和依赖关系。如果索引是从约束子句创建的，则需要构建pg_constraint条目，并将索引链接到约束，约束再链接到表。如果不是来自约束子句，则需要直接在索引和表之间创建依赖关系。这句话怎么理解呢？我的理解是如果是显示地使用create index创建索引，那么当原表被删除时，就需要级联删除这个索引，因此这里需要为索引注册一个对表地依赖。如果是从约束子句中自动创建的，那么索引直接依赖于这个依赖，当这个依赖被移除时，索引也要对应地被移除，所以这里创建一个pg_constraint条目，让索引依赖于它。

```c
	/*
	 * Register constraint and dependencies for the index.
	 *
	 * If the index is from a CONSTRAINT clause, construct a pg_constraint
	 * entry.  The index will be linked to the constraint, which in turn is
	 * linked to the table.  If it's not a CONSTRAINT, we need to make a
	 * dependency directly on the table.
	 *
	 * We don't need a dependency on the namespace, because there'll be an
	 * indirect dependency via our parent table.
	 *
	 * During bootstrap we can't register any dependencies, and we don't try
	 * to make a constraint either.
	 */
	if (!IsBootstrapProcessingMode())
	{
		ObjectAddress myself,
					referenced;
		ObjectAddresses *addrs;

		ObjectAddressSet(myself, RelationRelationId, indexRelationId);

		if ((flags & INDEX_CREATE_ADD_CONSTRAINT) != 0)
		{
			char		constraintType;
			ObjectAddress localaddr;
			...
```

## 5. 构建索引

构建索引内容如果当前处于bootstrap过程或者设置了INDEX_CREATE_SKIP_BUILD，填充索引的操作将被延迟到以后。如果没有上述情况，则调用索引访问方法（Access Method，AM）的例程来构建索引。

```c
	/*
	 * If this is bootstrap (initdb) time, then we don't actually fill in the
	 * index yet.  We'll be creating more indexes and classes later, so we
	 * delay filling them in until just before we're done with bootstrapping.
	 * Similarly, if the caller specified to skip the build then filling the
	 * index is delayed till later (ALTER TABLE can save work in some cases
	 * with this).  Otherwise, we call the AM routine that constructs the
	 * index.
	 */
	if (IsBootstrapProcessingMode())
	{
		index_register(heapRelationId, indexRelationId, indexInfo);
	}
	else if ((flags & INDEX_CREATE_SKIP_BUILD) != 0)
	{
		/*
		 * Caller is responsible for filling the index later on.  However,
		 * we'd better make sure that the heap relation is correctly marked as
		 * having an index.
		 */
		index_update_stats(heapRelation,
						   true,
						   -1.0);
		/* Make the above update visible */
		CommandCounterIncrement();
	}
	else
	{
		index_build(heapRelation, indexRelation, indexInfo, false, true);
	}

```
