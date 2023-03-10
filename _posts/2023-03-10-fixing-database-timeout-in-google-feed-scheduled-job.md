---
title: "Database timeout when running \"Google ProductFeed - Create feed\" scheduled job"
date: 2023-03-10
---

This blogpost addresses a timeout exception when running "Google ProductFeed - Create feed" scheduled job from GoogleProductFeed nuget package.

The issue is described by [itcotta] https://github.com/itcotta here: [#21] https://github.com/Geta/GoogleProductFeed/issues/21. While he fixed it by raising the number of DTU-s, for us, this was not a long-term solution. This was used on a huge catalog with several feeds that kept growing and we didn't want to endlessly upgrade the DB.

I am hereby copying the error message to improve searchability for the blogpost:

```Job failed. System.Data.Entity.Infrastructure.DbUpdateException: An error occurred while updating the entries. See the inner exception for details. ---> System.Data.Entity.Core.UpdateException: An error occurred while updating the entries. See the inner exception for details. ---> System.Data.SqlClient.SqlException: Execution Timeout Expired. The timeout period elapsed prior to completion of the operation or the server is not responding. The statement has been terminated. ---> System.ComponentModel.Win32Exception: The wait operation timed out --- End of inner exception stack trace --- at System.Data.SqlClient.SqlConnection.OnError(SqlException exception, Boolean breakConnection, Action1 wrapCloseInAction) at System.Data.SqlClient.TdsParser.ThrowExceptionAndWarning(TdsParserStateObject stateObj, Boolean callerHasConnectionLock, Boolean asyncClose) at System.Data.SqlClient.TdsParser.TryRun(RunBehavior runBehavior, SqlCommand cmdHandler, SqlDataReader dataStream, BulkCopySimpleResultSet bulkCopyHandler, TdsParserStateObject stateObj, Boolean& dataReady) at System.Data.SqlClient.SqlCommand.FinishExecuteReader(SqlDataReader ds, RunBehavior runBehavior, String resetOptionsString, Boolean isInternal, Boolean forDescribeParameterEncryption, Boolean shouldCacheForAlwaysEncrypted) at System.Data.SqlClient.SqlCommand.RunExecuteReaderTds(CommandBehavior cmdBehavior, RunBehavior runBehavior, Boolean returnStream, Boolean async, Int32 timeout, Task& task, Boolean asyncWrite, Boolean inRetry, SqlDataReader ds, Boolean describeParameterEncryptionRequest) at System.Data.SqlClient.SqlCommand.RunExecuteReader(CommandBehavior cmdBehavior, RunBehavior runBehavior, Boolean returnStream, String method, TaskCompletionSource1 completion, Int32 timeout, Task& task, Boolean& usedCache, Boolean asyncWrite, Boolean inRetry) at System.Data.SqlClient.SqlCommand.InternalExecuteNonQuery(TaskCompletionSource1 completion, String methodName, Boolean sendToPipe, Int32 timeout, Boolean& usedCache, Boolean asyncWrite, Boolean inRetry) at System.Data.SqlClient.SqlCommand.ExecuteNonQuery() at System.Data.Entity.Infrastructure.Interception.InternalDispatcher1.Dispatch[TTarget,TInterceptionContext,TResult](TTarget target, Func3 operation, TInterceptionContext interceptionContext, Action3 executing, Action3 executed) at System.Data.Entity.Infrastructure.Interception.DbCommandDispatcher.NonQuery(DbCommand command, DbCommandInterceptionContext interceptionContext) at System.Data.Entity.Core.Mapping.Update.Internal.DynamicUpdateCommand.Execute(Dictionary2 identifierValues, List1 generatedValues) at System.Data.Entity.Core.Mapping.Update.Internal.UpdateTranslator.Update() --- End of inner exception stack trace --- at System.Data.Entity.Core.Mapping.Update.Internal.UpdateTranslator.Update() at System.Data.Entity.Core.Objects.ObjectContext.ExecuteInTransaction[T](Func1 func, IDbExecutionStrategy executionStrategy, Boolean startLocalTransaction, Boolean releaseConnectionOnSuccess) at System.Data.Entity.Core.Objects.ObjectContext.SaveChangesToStore(SaveOptions options, IDbExecutionStrategy executionStrategy, Boolean startLocalTransaction) at System.Data.Entity.SqlServer.DefaultSqlExecutionStrategy.Execute[TResult](Func`1 operation) at System.Data.Entity.Core.Objects.ObjectContext.SaveChangesInternal(SaveOptions options, Boolean executeInExistingTransaction) at System.Data.Entity.Internal.InternalContext.SaveChanges() --- End of inner exception stack trace --- at System.Data.Entity.Internal.InternalContext.SaveChanges() at Geta.GoogleProductFeed.Repositories.FeedRepository.RemoveOldVersions(Int32 numberOfGeneratedFeeds) in C:\BuildAgent1\work\5931d0eced9e8a54\src\Geta.GoogleProductFeed\Repositories\FeedRepository.cs:line 42 at Geta.GoogleProductFeed.FeedHelper.GenerateAndSaveData() in C:\BuildAgent1\work\5931d0eced9e8a54\src\Geta.GoogleProductFeed\FeedHelper.cs:line 59 at Site.Infrastructure.GoogleMerchant.GoogleMerchantScheduledJob.Execute() in D:\buildAgent2\work\9009e16da83c5b64\src\Site\Infrastructure\GoogleMerchant\GoogleMerchantScheduledJob.cs:line 27
```

Instead, we figured out that we can easily inherit from IFeedRepository and implement RemoveOldVersions. There, we are passing the timeout from appSettings. The rest of the implementation is the same as in the library, so I am just copying the important bits here:

```
    public class CustomFeedRepository : IFeedRepository
    {
        private readonly int _timeout;
        private readonly FeedApplicationDbContext _applicationDbContext;
        
        public CustomFeedRepository(FeedApplicationDbContext applicationDbContext)
        {
            _applicationDbContext = applicationDbContext;
            _timeout = int.TryParse(ConfigurationManager.AppSettings["GoogleFeedBatchDeleteTimeout"], out var parsed) ? 
                parsed : 3 * 60;
        }

        public void RemoveOldVersions(int numberOfGeneratedFeeds)
        {
            var feedData = _applicationDbContext.FeedData;
            var count = feedData.Count();
            var deleted = feedData
                .OrderBy(x => x.CreatedUtc)
                .Take(count - numberOfGeneratedFeeds)
                .Delete(_timeout);
        }
    }
```

Please note that by the time you deploy this code to production (depending on the release cycle), the database entries might have piled up so much that you either need to extend the timeout or do the manual cleanup. After that, the code above should suffice.
