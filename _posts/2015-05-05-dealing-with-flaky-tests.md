---
ID: 257
post_title: Dealing with flaky tests using java.util.concurrent
author: Piotr Ciruk
post_excerpt: ""
layout: post
published: true
post_date: 2015-05-05 16:53:42
---
Whenever I hear about non-deterministic behaviour in software engineering a part of me dies inside. Non-deterministic tests are something to be ashamed of. A term 'flaky test' was coined to describe a situation when a test passes from time to time. This lack of determinism invalidates actual failures. Test does not pass? Let's run it again.

Flaky tests can be caused by many different things. One of the most frequent one is <a href="http://mir.cs.illinois.edu/~qluo2/fse14LuoHEM.pdf" target="_blank">async wait</a>. System under test involves some background processing or asynchronous call. Test is supposed to wait for this operation to finish and then run through the list of assertions.
The easiest way to achieve such result? Exploit the inglorious `Thread.sleep(int)` method. Ouch.
```
@Test
public void shouldCountVotesAndReturnAWinnerButIsFlaky() throws InterruptedException {
	Election election = new Election();

	election.countVotes();
	Thread.sleep(1000);
	
	assertThat(election.getWinner(), is(goodCandidate()));
}
```

```
static class Election {

	private Candidate winner;

	Candidate getWinner() {
		return winner;
	}

	void countVotes() {
		ExecutorService executorService = Executors.newSingleThreadExecutor();
		executorService.submit(() -> {
			collectVotesFromLocalHubs();
			selectAWinner();
		});
	}

	void selectAWinner() {
		winner = new Candidate();
	}

	void collectVotesFromLocalHubs() {
		for (int i = 0; i < 1e8; i++) {
			new Object().getClass();
		}

		System.out.println("Finished");
	}


}

static class Candidate {
}
```
	
Clearly, there are better ways of achieving the same goal. The one I deem the most obvious choice is usage of `java.util.concurrent` classes. The idea behind it is to create a wrapper of class which contains an asynchronous method. Create an object of it providing some kind of locking mechanism like `Condition` or `CountdownLatch`. Release the lock after the actual computation is finished. In the meantime the test method awaits this event to occur.
```
@Test
public void shouldCountVotesAndReturnAWinner() throws InterruptedException {
	CountDownLatch countDownLatch = new CountDownLatch(1);
	Election election = new Election() {
		CountDownLatch latch = countDownLatch;

		@Override
		void countVotes() {
			ExecutorService executorService = Executors.newSingleThreadExecutor();
			executorService.submit(() -> {
				collectVotesFromLocalHubs();
				selectAWinner();
				latch.countDown();
			});
		}
	};
	
	election.countVotes();
	countDownLatch.await(1, TimeUnit.SECONDS);

	assertThat(election.getWinner(), is(goodCandidate()));
}
```