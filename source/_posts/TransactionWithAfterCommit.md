---
title: PlatformTransactionManager&TransactionAwareCacheDecorator 源码解析+问题排查
date: 2023-2-13 14:41:31
tags:
- Java
- Spring
categories:
- Spring
---
# PlatformTransactionManager&TransactionAwareCacheDecorator 源码解析+问题排查
## 1.问题现象
	在原有使用中发现原有删除缓存的操作未能正常执行。原code 如下:
	```
	@Transactional
	void function{
	db_action();
	db_action2();
	//在上方事务提交后，进行缓存和通知其他service动作
	TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
            @Override
            public void afterCommit() {
            	    deleteCache();
                action2();
            }
        })   
    }
	```
   排查思路：
   1.确认afterCommit 是否被调用
   	发现被调用，但是查看缓存后没有被执行
   2. 了解afterCommit如何被处理
   3. 尝试debug transactionManager 执行部分Code，确定整体流程
   <!-- more -->
   根据debug和阅读代码可以发现
   TransactionSynchronizationManager 中存储了所有当前的transaction信息，registerSynchronization就会把匿名类加入到set中。
   private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
		new NamedThreadLocal<>("Transaction synchronizations");
			
	根据synchronizations.get()被调用方法可以找到AbstractPlatformTransactionManager.processCommit()会对事务进行处理，包括before 和 after等操作。code如下：

	```
	/**
	 * Process an actual commit.
	 * Rollback-only flags have already been checked and applied.
	 * @param status object representing the transaction
	 * @throws TransactionException in case of commit failure
	 */
	private void processCommit(DefaultTransactionStatus status) throws TransactionException {
		try {
			boolean beforeCompletionInvoked = false;

			try {
				boolean unexpectedRollback = false;
				prepareForCommit(status);
				// 处理beforeCommit方法
				triggerBeforeCommit(status);
				triggerBeforeCompletion(status);
				beforeCompletionInvoked = true;

				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Releasing transaction savepoint");
					}
					unexpectedRollback = status.isGlobalRollbackOnly();
					status.releaseHeldSavepoint();
				}
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction commit");
					}
					unexpectedRollback = status.isGlobalRollbackOnly();
					doCommit(status);
				}
				else if (isFailEarlyOnGlobalRollbackOnly()) {
					unexpectedRollback = status.isGlobalRollbackOnly();
				}

				// Throw UnexpectedRollbackException if we have a global rollback-only
				// marker but still didn't get a corresponding exception from commit.
				if (unexpectedRollback) {
					throw new UnexpectedRollbackException(
							"Transaction silently rolled back because it has been marked as rollback-only");
				}
			}
			catch (UnexpectedRollbackException ex) {
				// can only be caused by doCommit
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
				throw ex;
			}
			catch (TransactionException ex) {
				// can only be caused by doCommit
				if (isRollbackOnCommitFailure()) {
					doRollbackOnCommitException(status, ex);
				}
				else {
					triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				}
				throw ex;
			}
			catch (RuntimeException | Error ex) {
				if (!beforeCompletionInvoked) {
					triggerBeforeCompletion(status);
				}
				doRollbackOnCommitException(status, ex);
				throw ex;
			}

			// Trigger afterCommit callbacks, with an exception thrown there
			// propagated to callers but the transaction still considered as committed.
			try {
				triggerAfterCommit(status);
			}
			finally {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
			}

		}
		finally {
			cleanupAfterCompletion(status);
		}
	}
	```