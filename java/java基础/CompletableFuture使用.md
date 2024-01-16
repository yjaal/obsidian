
这里一般有两种使用方式，一种是需要返回值，一种是不需要返回值的

## 无需返回值

```java
List<CompletableFuture<Void>> markFutures = new ArrayList<>(pageSize);
	for (List<TagExternalUserInfoEntity> needDealTagUserInfos : tagUserInfosList) {
		for (TagExternalUserInfoEntity tagUserInfo : needDealTagUserInfos) {
			CompletableFuture<Void> markFuture = 
			CompletableFuture.runAsync(() -> 
				markLabelDetail(tagUserInfo, secret), pool);
				
			markFutures.add(markFuture);
		}
		// 异步转同步，等待
		CompletableFuture.allOf(markFutures.toArray(new CompletableFuture[markFutures.size()])).get();
	}
```

这里我们可以传入自定义的线程池，当然也可以不传。异步转同步使用的是 get 方法

## 需要返回值

```java
List<CompletableFuture<Integer>> countFutures =
	userIdList.parallelStream()
			.map(id -> CompletableFuture.supplyAsync(
					() -> externalUserFollowDao.countExternalUserByUserIdList(corpId, agentId,
							Collections.singletonList(id), DateUtils.addDays(date, 1))))
			.collect(Collectors.toList());

List<CompletableFuture<Integer>> countDelFutures =
	userIdList.parallelStream()
			.map(id -> CompletableFuture.supplyAsync(
					() -> externalUserFollowDao.countDelExtUserByUserIdList(corpId, agentId,
							Collections.singletonList(id), DateUtils.addDays(date, 1))))
			.collect(Collectors.toList());
countFutures.addAll(countDelFutures);
CompletableFuture<Void> allOf = CompletableFuture.allOf(countFutures.toArray(new CompletableFuture[0]));
allOf.join();
for (CompletableFuture<Integer> countFuture : countFutures) {
cnt += countFuture.get();
}
```

这里没有传入自定义的线程池，同时异步转同步使用的是 join 方法，其与 get 方法的作用是一样的，只是 get 方法会同步阻塞抛出异常，而 join 是非阻塞的，其异常可以通过最后的在取每一个返回值 get 方法时获得。