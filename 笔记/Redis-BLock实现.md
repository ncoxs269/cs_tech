2025-05-01 16:35
Status: #idea
Tags: [[Redis]]


# 1 文件 errors  
```Go  
package block  
  
import (  
    "errors"  
    "fmt"  
)  
  
var (  
    ErrRenewTTL             = errors.New("BLock renew ttl less than 1s error")  
    ErrRedisClientIsNil     = errors.New("BLock redis client is nil error")  
    ErrNotLockOwner         = errors.New("BLock can't handle redLock of other routine error")  
    ErrNotExitLock          = errors.New("BLock is not exit error")  
    ErrUnknown              = errors.New("BLock unknown error")  
    ErrNoMarshalLock        = errors.New("BLock can not marshall reliable or long lease lock error")  
    ErrTimeAndRoundConflict = errors.New("BLock timeout and round parameters are conflict")  
)  
  
type ErrCronJob struct {  
    CronJobName string  
}  
  
func (e ErrCronJob) Error() string {  
    return "BLock " + e.CronJobName + " Cron job error"  
}  
  
type ErrCronJobUnfinished struct {  
    CronJobName string  
}  
  
func (e ErrCronJobUnfinished) Error() string {  
    return "BLock " + e.CronJobName + " Cron job didn't finish last round"  
}  
  
func wrapBLockErr(err error) error {  
    return fmt.Errorf("BLock error %w", err)  
}  
```  
  
# 2 文件 redlock  
```Go  
package block  
  
import (  
    "bytes"  
    "context"  
    "crypto/rand"  
    "encoding/json"  
    "math/big"  
    "strconv"  
    "sync"  
    "sync/atomic"  
    "time"  
  
    "code.byted.org/gopkg/logs"  
    "code.byted.org/kv/goredis"  
    "code.byted.org/kv/redis-v6/pkg"  
    "github.com/google/uuid"  
    cmap "github.com/orcaman/concurrent-map"  
    "github.com/pkg/errors"  
    "github.com/robfig/cron/v3"  
)  
  
const (  
    // LongLeaseLockDuration define the lower bound of lock duration.  
    LongLeaseLockDuration = 30 * time.Second  
  
    // ReliableLockDuration is default duration time if locked reliable.  
    ReliableLockDuration = 30 * time.Second  
  
    // WatchDogInterval is an expression used in watch dog cron job which manage redLock reliable.  
    WatchDogInterval = "@every 10s"  
  
    // LongLeaseLockCronInterval is an expression used in long lease lock cron job which manage long lock.  
    LongLeaseLockCronInterval = "@every 60s"  
  
    // HandleBatch is the count of every go runtime could handle.  
    HandleBatch = 100  
  
    // RandomSleepInterval is sleep interval time between lock.  
    RandomSleepInterval = 5  
  
    // LongLeaseLockCronJob is name of long lease lock cron job  
    LongLeaseLockCronJob = "LongLeaseLock"  
  
    // WatchDogCronJob is name of watch dog cron job  
    WatchDogCronJob = "WatchDog"  
  
    ZeroDuration = 0 * time.Millisecond  
  
    redisReadWriteTimeout = 500 * time.Millisecond  
)  
  
var (  
    lockCronJobList = []*LockCronJob{  
        {  
            Cron:         cron.New(),  
            IntervalExpr: LongLeaseLockCronInterval,  
            Cmap:         longLeaseLockMap,  
            RunningFlag:  &RunningFlag{false},  
            JobName:      LongLeaseLockCronJob,  
            Handler:      longLeaseLockHandle,  
        },  
        {  
            Cron:         cron.New(),  
            IntervalExpr: WatchDogInterval,  
            Cmap:         watchDogMap,  
            RunningFlag:  &RunningFlag{false},  
            JobName:      WatchDogCronJob,  
            Handler:      reliableLockHandle,  
        },  
    }  
  
    // watchDogMap store unreliable redLocks, support concurrent access.  
    longLeaseLockMap = cmap.New()  
  
    // watchDogMap store reliable redLocks, support concurrent access.  
    watchDogMap = cmap.New()  
  
    // serPSM records PSM of service.  
    serPSM string  
  
    // serPSM records PSM of redis cluster.  
    redPSM string  
  
    // client is client of goredis, should be initialized when service init.  
    client *goredis.Client  
)  
  
type RedLock struct {  
    target        string  
    redisKey      string  
    redisValue    string  
    reliable      bool  
    longLeaseLock bool  
    timeToLeave   time.Time  
}  
  
// NewLock init a redLock, users don't need to pay attention  
// at the key and value of the lock that are stored in redLock.  
func NewLock(dest string) (RedLock, error) {  
    var redLock RedLock  
  
    newUUID, err := uuid.NewUUID()  
    if err != nil {  
        return redLock, wrapBLockErr(err)  
    }  
  
    redLock = RedLock{  
        target:     dest,  
        redisKey:   RedLockKey(dest),  
        redisValue: newUUID.String(),  
        reliable:   false,  
    }  
  
    return redLock, nil  
}  
  
// initClient initialize a unique redis client.  
func initClient(redisPSM string) error {  
    redPSM = redisPSM  
    opt := goredis.NewOption()  
    opt.SetServiceDiscoveryWithConsul()  
    opt.SetMaxRetries(0)  
    opt.WriteTimeout = redisReadWriteTimeout  
    opt.ReadTimeout = redisReadWriteTimeout  
  
    cli, err := goredis.NewClientWithOption(redPSM, opt)  
    if err != nil {  
        return errors.WithMessage(err, "BLock init redis client error")  
    }  
  
    client = cli  
  
    return nil  
}  
  
// Init initialize BLock environment, and should be called as application starts.  
// servicePSM is PSM of service registered in ByteDance  
// redisPSM is PSM of redis cluster registered in ByteDance  
func Init(ctx context.Context, servicePSM, redisPSM string) error {  
    serPSM = servicePSM  
  
    if err := initClient(redisPSM); err != nil {  
        return err  
    }  
  
    for _, job := range lockCronJobList {  
        if err := initCronJob(ctx, job); err != nil {  
            return err  
        }  
    }  
  
    return nil  
}  
  
// InitWithCli initialize BLock environment, and should be called as application starts.  
// servicePSM is PSM of service registered in ByteDance  
// cli is redis client  
func InitWithCli(ctx context.Context, servicePSM string, cli *goredis.Client) error {  
    serPSM = servicePSM  
  
    if cli == nil {  
        return ErrRedisClientIsNil  
    }  
  
    client = cli  
  
    for _, job := range lockCronJobList {  
        if err := initCronJob(ctx, job); err != nil {  
            return err  
        }  
    }  
  
    return nil  
}  
  
func GracefulUnlock(ctx context.Context) {  
    for _, job := range lockCronJobList {  
        lockCount := job.Cmap.Count()  
        unlockCount := batchUnlock(ctx, HandleBatch, job.Cmap)  
        logs.CtxDebug(ctx, "[BLock/GracefulUnlock] graceful unlock %v: total=%v, "+  
            "unlock=%v", job.JobName, lockCount, unlockCount)  
    }  
}  
  
func batchUnlock(ctx context.Context, batchNum int, lockMap cmap.ConcurrentMap) int {  
    var counter uint64  
  
    ch := lockMap.IterBuffered()  
  
    goNum := cap(ch)/batchNum + 1  
  
    var wg sync.WaitGroup  
  
    wg.Add(goNum)  
  
    for i := 0; i < goNum; i++ {  
        go func() {  
            defer wg.Done()  
  
            for item := range ch {  
                redisKey := item.Key  
                redLock, _ := item.Val.(*RedLock)  
  
                if err := redLock.UnLock(ctx); err != nil {  
                    logs.CtxWarn(ctx, "[BLock/GracefulUnlock] unlock fail: %v, key=%v, value=%v", err, redisKey, redLock.redisValue)  
                } else {  
                    atomic.AddUint64(&counter, 1)  
                }  
            }  
        }()  
    }  
  
    wg.Wait()  
  
    return int(atomic.LoadUint64(&counter))  
}  
  
// TryLock will try to lock resource once, returns true if lock success, otherwise returns false.  
// ttl is the time to release the lock in milliSeconds.  
// If ttl less than or equal to 0, redLock will handle the life circle of the lock.  
func (redLock *RedLock) TryLock(ctx context.Context, ttl time.Duration) (bool, error) {  
    if ttl <= 0 {  
        redLock.reliable = true  
        ttl = ReliableLockDuration  
    } else if ttl >= LongLeaseLockDuration {  
        redLock.longLeaseLock = true  
    }  
  
    client, err := getClient(ctx)  
    if err != nil {  
        return false, err  
    }  
  
    result, err := client.SetNX(redLock.redisKey, redLock.redisValue, ttl).Result()  
    if pkg.IsNetworkError(err) {  
        result, err = redLock.retryLock(client, ttl)  
    }  
  
    if err != nil {  
        return false, wrapBLockErr(err)  
    }  
  
    if !result {  
        return false, nil  
    }  
  
    redLock.timeToLeave = time.Now().Add(ttl)  
  
    if redLock.reliable {  
        watchDogMap.Set(redLock.redisKey, redLock)  
    }  
  
    if redLock.longLeaseLock {  
        longLeaseLockMap.Set(redLock.redisKey, redLock)  
    }  
  
    return result, nil  
}  
  
func (redLock *RedLock) LockWithInterval(ctx context.Context, ttl time.Duration,  
    round int, interval time.Duration) (bool, error) {  
    return redLock.lockRound(ctx, ttl, round, interval)  
}  
  
// Lock will try to lock resource within given time or round.  
// Returns true if lock success, otherwise returns false.  
// ttl is the time to release the lock. If ttl less than or equal to 0, redLock will handle the life circle of the lock.  
// timeout is the given time.  
// round is the specified round.  
func (redLock *RedLock) Lock(ctx context.Context, ttl time.Duration, timeout time.Duration, round int) (bool, error) {  
    result := false  
  
    var err error  
  
    if timeout.Milliseconds()*int64(round) > 0 || timeout.Milliseconds() < 0 || round < 0 ||  
        (timeout.Milliseconds() == 0 && round == 0) {  
        return false, ErrTimeAndRoundConflict  
    }  
  
    if ttl <= 0 {  
        redLock.reliable = true  
        ttl = ReliableLockDuration  
    } else if ttl >= LongLeaseLockDuration {  
        redLock.longLeaseLock = true  
    }  
  
    if timeout > 0 {  
        result, err = redLock.lockTimed(ctx, ttl, timeout)  
    } else if round > 0 {  
        result, err = redLock.lockRound(ctx, ttl, round, ZeroDuration)  
    }  
  
    if !result || err != nil {  
        return false, err  
    }  
  
    redLock.timeToLeave = time.Now().Add(ttl)  
  
    if redLock.reliable {  
        watchDogMap.Set(redLock.redisKey, redLock)  
    }  
  
    if redLock.longLeaseLock {  
        longLeaseLockMap.Set(redLock.redisKey, redLock)  
    }  
  
    return result, err  
}  
  
// lockTimed will try to lock resource within a given time, returns true if lock success, otherwise returns false.  
// ttl is the time to release the lock.  
// timeout is the given time.  
func (redLock *RedLock) lockTimed(ctx context.Context, ttl time.Duration, timeout time.Duration) (bool, error) {  
    client, err := getClient(ctx)  
    if err != nil {  
        return false, err  
    }  
  
    end := time.Now().Add(timeout)  
    for time.Now().Before(end) {  
        result, err := client.SetNX(redLock.redisKey, redLock.redisValue, ttl).Result()  
        if pkg.IsNetworkError(err) {  
            result, err = redLock.retryLock(client, ttl)  
        }  
  
        if err != nil {  
            return false, wrapBLockErr(err)  
        }  
  
        if result {  
            return true, nil  
        }  
  
        interval, err := getRandomInterval()  
        if err != nil {  
            return false, err  
        }  
  
        time.Sleep(interval)  
    }  
  
    return false, nil  
}  
  
// lockRound will try to lock resource in a specified round, returns true if lock success, otherwise returns false.  
// ttl is the time to release the lock.  
// round is the specified round.  
func (redLock *RedLock) lockRound(ctx context.Context, ttl time.Duration,  
    round int, interval time.Duration) (bool, error) {  
    client, err := getClient(ctx)  
    if err != nil {  
        return false, err  
    }  
  
    for i := 0; i < round; i++ {  
        result, err := client.SetNX(redLock.redisKey, redLock.redisValue, ttl).Result()  
        if pkg.IsNetworkError(err) {  
            result, err = redLock.retryLock(client, ttl)  
        }  
  
        if err != nil {  
            return false, wrapBLockErr(err)  
        }  
  
        if result {  
            return true, nil  
        }  
  
        if interval <= 0 {  
            interval, err = getRandomInterval()  
            if err != nil {  
                return false, err  
            }  
        }  
  
        time.Sleep(interval)  
    }  
  
    return false, nil  
}  
  
func (redLock *RedLock) retryLock(client *goredis.Client, ttl time.Duration) (bool, error) {  
    result, err := client.SetNX(redLock.redisKey, redLock.redisValue, ttl).Result()  
    if err != nil {  
        return false, wrapBLockErr(err)  
    }  
  
    if result {  
        return true, nil  
    }  
  
    val, err := client.Get(redLock.redisKey).Result()  
    if err != nil {  
        return false, wrapBLockErr(err)  
    }  
  
    if val == redLock.redisValue {  
        return true, nil  
    }  
  
    return false, nil  
}  
  
// Renew is used to extend the lifetime of the lock.  
// ttl is the time to release the lock in milliseconds.  
func (redLock *RedLock) Renew(ctx context.Context, ttl time.Duration) (bool, error) {  
    client, err := getClient(ctx)  
    if err != nil {  
        return false, err  
    }  
  
    if ttl > 0 && ttl < time.Second {  
        return false, ErrRenewTTL  
    }  
  
    result, err := client.WithContext(ctx).  
        CasEx(redLock.redisKey, redLock.redisValue, redLock.redisValue, ttl).  
        Result()  
    if err != nil {  
        return false, wrapBLockErr(err)  
    }  
  
    switch result {  
    case 1:  
        redLock.timeToLeave = time.Now().Add(ttl)  
  
        if ttl >= LongLeaseLockDuration && !redLock.longLeaseLock {  
            redLock.longLeaseLock = true  
            longLeaseLockMap.Set(redLock.redisKey, redLock)  
        }  
  
        return true, nil  
    case -1:  
        return false, ErrNotExitLock  
    case 0:  
        return false, ErrNotLockOwner  
    }  
  
    return false, ErrUnknown  
}  
  
// UnLock can release the lock, returns error if unlock exception.  
func (redLock *RedLock) UnLock(ctx context.Context) error {  
    if redLock.reliable {  
        watchDogMap.Remove(redLock.redisKey)  
    }  
  
    if redLock.longLeaseLock {  
        longLeaseLockMap.Remove(redLock.redisKey)  
    }  
  
    client, err := getClient(ctx)  
    if err != nil {  
        return err  
    }  
  
    result, err := client.WithContext(ctx).Cad(redLock.redisKey, redLock.redisValue).Result()  
    if err != nil {  
        return wrapBLockErr(err)  
    }  
  
    switch result {  
    case 1:  
        return nil  
    case -1:  
        return ErrNotExitLock  
    case 0:  
        return ErrNotLockOwner  
    default:  
        return ErrUnknown  
    }  
}  
  
// RedLockKey generates the key of the lock store in redis cluster.  
// dest is the resource to be locked.  
func RedLockKey(dest string) string {  
    return serPSM + "_" + dest  
}  
  
func getClient(ctx context.Context) (*goredis.Client, error) {  
    if client == nil {  
        return nil, ErrRedisClientIsNil  
    }  
  
    return client.WithContext(ctx), nil  
}  
  
func getRandomInterval() (time.Duration, error) {  
    interval, err := rand.Int(rand.Reader, new(big.Int).SetInt64(int64(RandomSleepInterval)))  
    if err != nil {  
        return RandomSleepInterval, wrapBLockErr(err)  
    }  
  
    sleepDuration, err := time.ParseDuration(strconv.FormatInt(interval.Int64(), 10) + "ms")  
    if err != nil {  
        return RandomSleepInterval, wrapBLockErr(err)  
    }  
  
    return sleepDuration, nil  
}  
  
func Marshal(redLock *RedLock) ([]byte, error) {  
    if redLock.reliable || redLock.longLeaseLock {  
        return nil, ErrNoMarshalLock  
    }  
  
    attrMap := make(map[string]interface{}, 6)  
    attrMap["target"] = redLock.target  
    attrMap["redisKey"] = redLock.redisKey  
    attrMap["redisValue"] = redLock.redisValue  
    attrMap["reliable"] = redLock.reliable  
    attrMap["longLeaseLock"] = redLock.longLeaseLock  
    attrMap["timeToLeave"] = redLock.timeToLeave.Unix()  
  
    buf, err := json.Marshal(attrMap)  
    if err != nil {  
        return nil, wrapBLockErr(err)  
    }  
  
    return buf, nil  
}  
  
func Unmarshal(data []byte) (RedLock, error) {  
    var attrMap map[string]interface{}  
  
    var redLock RedLock  
  
    decoder := json.NewDecoder(bytes.NewReader(data))  
    decoder.UseNumber()  
  
    if err := decoder.Decode(&attrMap); err != nil {  
        return redLock, wrapBLockErr(err)  
    }  
  
    ttl, _ := attrMap["timeToLeave"].(json.Number).Int64()  
  
    redLock = RedLock{  
        target:        attrMap["target"].(string),  
        redisKey:      attrMap["redisKey"].(string),  
        redisValue:    attrMap["redisValue"].(string),  
        reliable:      attrMap["reliable"].(bool),  
        longLeaseLock: attrMap["longLeaseLock"].(bool),  
        timeToLeave:   time.Unix(ttl, 1e9),  
    }  
  
    return redLock, nil  
}  
  
func (redLock *RedLock) LockTarget() string {  
    return redLock.target  
}  
  
func (redLock *RedLock) LockKey() string {  
    return redLock.redisKey  
}  
  
func (redLock *RedLock) LockValue() string {  
    return redLock.redisValue  
}  
```  
  
# 3 文件 cron  
```Go  
package block  
  
import (  
    "context"  
    "sync"  
    "sync/atomic"  
    "time"  
  
    "code.byted.org/gopkg/logs"  
    cmap "github.com/orcaman/concurrent-map"  
    "github.com/robfig/cron/v3"  
)  
  
// RunningFlag identify ths status of the cron job,  
// returns false if previous round of cron job has not finished.  
type RunningFlag struct {  
    isRunning bool  
}  
  
func (runningFlag *RunningFlag) setFlag(flag bool) {  
    runningFlag.isRunning = flag  
}  
  
// LockHandler will handle redLocks in the cron job.  
type LockHandler func(ctx context.Context, redLock *RedLock, counter *uint64) error  
  
type LockCronJob struct {  
    // Cron is a timer task to renew redLocks that are reliable.  
    Cron *cron.Cron  
  
    // IntervalExpr is the time expression used in cron job.  
    IntervalExpr string  
  
    // Cmap stores redLocks, support concurrent access.  
    Cmap cmap.ConcurrentMap  
  
    // RunningFlag identify ths status of the cron job,  
    // returns false if previous round of cron job has not finished.  
    RunningFlag *RunningFlag  
  
    // JobName is the name of cron job.  
    JobName string  
  
    // Handler will handle redLocks in the cron job.  
    Handler LockHandler  
}  
  
func initCronJob(ctx context.Context, lockCronJob *LockCronJob) error {  
    var err error  
  
    _, err = lockCronJob.Cron.AddFunc(lockCronJob.IntervalExpr, func() {  
        defer func() {  
            if r := recover(); r != nil {  
                err := ErrCronJob{lockCronJob.JobName}  
                logs.CtxError(ctx, "[BLock/initCronJob] error: %v", err)  
            }  
        }()  
  
        if lockCronJob.Cmap.Count() == 0 {  
            logs.CtxDebug(ctx, "[BLock/initCronJob] %v is nil", lockCronJob.JobName)  
  
            return  
        }  
  
        if lockCronJob.RunningFlag.isRunning {  
            err := ErrCronJobUnfinished{lockCronJob.JobName}  
            logs.CtxWarn(ctx, "[BLock/initCronJob] %v", err)  
  
            return  
        }  
  
        count := runJob(ctx, lockCronJob)  
        logs.CtxDebug(ctx, "[BLock/initCronJob] %v Cron job succeed: handle=%v, "+  
            "mapLen=%v", lockCronJob.JobName, count, lockCronJob.Cmap.Count())  
    })  
  
    if err != nil {  
        return wrapBLockErr(err)  
    }  
  
    lockCronJob.Cron.Start()  
  
    return nil  
}  
  
func runJob(ctx context.Context, lockCronJob *LockCronJob) int {  
    lockCronJob.RunningFlag.setFlag(true)  
  
    var counter uint64  
  
    ch := lockCronJob.Cmap.IterBuffered()  
  
    goNum := cap(ch)/HandleBatch + 1  
  
    var wg sync.WaitGroup  
  
    wg.Add(goNum)  
  
    for i := 0; i < goNum; i++ {  
        go func() {  
            defer wg.Done()  
  
            for item := range ch {  
                redLock, _ := item.Val.(*RedLock)  
                if err := lockCronJob.Handler(ctx, redLock, &counter); err != nil {  
                    logs.CtxWarn(ctx, "[BLock/runJob] %v", err)  
                }  
            }  
        }()  
    }  
  
    wg.Wait()  
  
    lockCronJob.RunningFlag.setFlag(false)  
  
    return int(atomic.LoadUint64(&counter))  
}  
  
func reliableLockHandle(ctx context.Context, redLock *RedLock, counter *uint64) error {  
    client, err := getClient(ctx)  
    if err != nil {  
        return err  
    }  
  
    key := redLock.redisKey  
    val := redLock.redisValue  
  
    result, err := client.WithContext(ctx).CasEx(key, val, val, ReliableLockDuration).Result()  
    if err != nil {  
        return wrapBLockErr(err)  
    }  
  
    atomic.AddUint64(counter, 1)  
  
    if result == -1 && watchDogMap.Has(key) {  
        watchDogMap.Remove(key)  
  
        return ErrNotExitLock  
    } else if result == 0 && watchDogMap.Has(key) {  
        watchDogMap.Remove(key)  
  
        return ErrNotLockOwner  
    }  
  
    return nil  
}  
  
func longLeaseLockHandle(ctx context.Context, redLock *RedLock, counter *uint64) error {  
    now := time.Now()  
    if redLock.timeToLeave.Before(now) && longLeaseLockMap.Has(redLock.redisKey) {  
        longLeaseLockMap.Remove(redLock.redisKey)  
        logs.CtxDebug(ctx, "[BLock/longLeaseLockHandle] expire lock:, key=%v, value=%v",  
            redLock.redisKey, redLock.redisValue)  
  
        atomic.AddUint64(counter, 1)  
    }  
  
    return nil  
}  
```  
  
# 4 文件 redlock_test  
```Go  
package block  
  
import (  
    "context"  
    "encoding/json"  
    "errors"  
    "strconv"  
    "sync"  
    "sync/atomic"  
    "testing"  
    "time"  
  
    "code.byted.org/gopkg/logs"  
    . "code.byted.org/gopkg/mockito"  
    "code.byted.org/kv/goredis"  
    . "github.com/smartystreets/goconvey/convey"  
)  
  
// 两个集群都是boe环境有效集群，均可使用  
// const BOERedisPSM = "bytedance.redis.blockbenchmark"  
const BOERedisPSM = "toutiao.redis.infra_devops"  
  
func TestNewLock(t *testing.T) {  
    type args struct {  
        dest string  
    }  
    type testConfig struct {  
        args    args  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        err := initClient(BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    dest: "lockKey",  
                },  
                wantErr: false,  
            }  
            _, err := NewLock(tt.args.dest)  
            So(err != nil, ShouldEqual, tt.wantErr)  
        })  
    })  
}  
  
func Test_init(t *testing.T) {  
    type args struct {  
        ctx        context.Context  
        servicePSM string  
        redisPSM   string  
    }  
    type testConfig struct {  
        args    args  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:        context.Background(),  
                    servicePSM: "BLockTest",  
                    redisPSM:   BOERedisPSM,  
                },  
                wantErr: false,  
            }  
            err := Init(tt.args.ctx, tt.args.servicePSM, tt.args.redisPSM)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            redLock, _ := NewLock("Init")  
            got, _ := redLock.TryLock(tt.args.ctx, 0)  
            So(got, ShouldEqual, true)  
            value, ok := watchDogMap.Get(redLock.redisKey)  
            So(ok, ShouldEqual, true)  
            mapLock, ok := value.(*RedLock)  
            So(ok, ShouldEqual, true)  
  
            client, _ := getClient(tt.args.ctx)  
            value1 := client.Get(redLock.redisKey).Val()  
            So(value1, ShouldEqual, redLock.redisValue)  
            So(mapLock.redisValue, ShouldEqual, redLock.redisValue)  
  
            time.Sleep(time.Duration(5) * time.Second)  
  
            got2, _ := redLock.TryLock(tt.args.ctx, 10*time.Second)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got2, ShouldEqual, false)  
  
            err = redLock.UnLock(tt.args.ctx)  
            So(err == nil, ShouldEqual, true)  
            _, ok = watchDogMap.Get(redLock.redisKey)  
            So(ok, ShouldEqual, false)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:        context.Background(),  
                    servicePSM: "BLockTest",  
                    redisPSM:   "",  
                },  
                wantErr: false,  
            }  
            err := Init(tt.args.ctx, tt.args.servicePSM, tt.args.redisPSM)  
            So(err == nil, ShouldEqual, tt.wantErr)  
        })  
    })  
}  
  
func TestInitWithCli(t *testing.T) {  
    type args struct {  
        ctx        context.Context  
        servicePSM string  
        cli        *goredis.Client  
    }  
    type testConfig struct {  
        args    args  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:        context.Background(),  
                    servicePSM: "service",  
                    cli:        nil,  
                },  
                wantErr: true,  
            }  
            err := InitWithCli(tt.args.ctx, tt.args.servicePSM, tt.args.cli)  
            So(err != nil, ShouldEqual, tt.wantErr)  
        })  
        PatchConvey("test case2", func() {  
            opt := goredis.NewOption()  
            opt.SetServiceDiscoveryWithConsul()  
            opt.SetMaxRetries(0)  
  
            cli, _ := goredis.NewClientWithOption(BOERedisPSM, opt)  
  
            tt := testConfig{  
                args: args{  
                    ctx:        context.Background(),  
                    servicePSM: "service",  
                    cli:        cli,  
                },  
                wantErr: false,  
            }  
            err := InitWithCli(tt.args.ctx, tt.args.servicePSM, tt.args.cli)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            redLock, _ := NewLock("InitWithCli")  
            got, _ := redLock.TryLock(tt.args.ctx, 0)  
            So(got, ShouldEqual, true)  
            value, ok := watchDogMap.Get(redLock.redisKey)  
            So(ok, ShouldEqual, true)  
            mapLock, ok := value.(*RedLock)  
            So(ok, ShouldEqual, true)  
  
            client, _ := getClient(tt.args.ctx)  
            value1 := client.Get(redLock.redisKey).Val()  
            So(value1, ShouldEqual, redLock.redisValue)  
            So(mapLock.redisValue, ShouldEqual, redLock.redisValue)  
  
            got2, _ := redLock.TryLock(tt.args.ctx, 10*time.Second)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got2, ShouldEqual, false)  
  
            err = redLock.UnLock(tt.args.ctx)  
            So(err == nil, ShouldEqual, true)  
            _, ok = watchDogMap.Get(redLock.redisKey)  
            So(ok, ShouldEqual, false)  
        })  
    })  
}  
  
func TestRedLock_TryLock(t *testing.T) {  
    type args struct {  
        ctx  context.Context  
        dest string  
        ttl  time.Duration  
    }  
    type testConfig struct {  
        args      args  
        want      bool  
        wantErr   bool  
        wantToken bool  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:  context.Background(),  
                    dest: "lockDest",  
                    ttl:  30 * time.Second,  
                },  
                wantErr:   false,  
                want:      true,  
                wantToken: true,  
            }  
            err := initClient(BOERedisPSM)  
            if err != nil {  
                return  
            }  
            redLock, _ := NewLock("TyrLock")  
            got, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            duration := client.TTL(redLock.redisKey)  
            logs.Info("[BLock/TestRedLock_TryLock] redisKey=%v, redisValue=%v, duration=%v", redLock.redisKey, redLock.redisValue, duration.Val())  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                wantErr: false,  
            }  
            err := initClient("")  
            So(err == nil, ShouldEqual, tt.wantErr)  
        })  
    })  
}  
  
func TestRedLock_LockWithInterval(t *testing.T) {  
    type args struct {  
        ctx      context.Context  
        ttl      time.Duration  
        round    int  
        interval time.Duration  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        err := initClient(BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:      context.Background(),  
                    ttl:      30 * time.Second,  
                    round:    10,  
                    interval: 200 * time.Millisecond,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("LockWithInterval")  
            got, err := redLock.LockWithInterval(tt.args.ctx, tt.args.ttl, tt.args.round, tt.args.interval)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
    })  
}  
  
func TestRedLock_Lock(t *testing.T) {  
    type args struct {  
        ctx     context.Context  
        ttl     time.Duration  
        timeout time.Duration  
        round   int  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     30 * time.Second,  
                    round:   10,  
                    timeout: 0,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            client = nil  
            err := initClient("")  
            So(err == nil, ShouldEqual, tt.wantErr)  
            redLock, _ := NewLock("Lock1")  
            _, err = redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
            So(err == nil, ShouldEqual, tt.wantErr)  
            _ = redLock.UnLock(tt.args.ctx)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     0,  
                    round:   30,  
                    timeout: 0,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            client = nil  
            err := initClient(BOERedisPSM)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            redLock, _ := NewLock("lock2")  
            got, err := redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
  
            So(err == nil, ShouldBeTrue)  
            So(got, ShouldEqual, tt.want)  
            _ = redLock.UnLock(tt.args.ctx)  
        })  
        PatchConvey("test case3", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     0,  
                    round:   0,  
                    timeout: 0,  
                },  
                wantErr: true,  
                want:    false,  
            }  
            client = nil  
            err := initClient(BOERedisPSM)  
            So(err == nil, ShouldBeTrue)  
  
            redLock, _ := NewLock("lock3")  
            got, err := redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case4", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     0,  
                    round:   10,  
                    timeout: 10 * time.Millisecond,  
                },  
            }  
            client = nil  
            err := initClient(BOERedisPSM)  
            So(err == nil, ShouldBeTrue)  
  
            redLock, _ := NewLock("lock4")  
            _, err = redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
  
            So(err != nil, ShouldBeTrue)  
        })  
        PatchConvey("test case5", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     0,  
                    round:   -10,  
                    timeout: -10 * time.Millisecond,  
                },  
                wantErr: true,  
                want:    false,  
            }  
            client = nil  
            err := initClient(BOERedisPSM)  
            So(err == nil, ShouldBeTrue)  
  
            redLock, _ := NewLock("lock5")  
            _, err = redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
  
            So(err != nil, ShouldBeTrue)  
        })  
  
    })  
}  
  
func TestRedLock_LockTimed(t *testing.T) {  
    type args struct {  
        ctx     context.Context  
        ttl     time.Duration  
        timeout time.Duration  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                },  
                wantErr: false,  
                want:    true,  
            }  
            client = nil  
            err := initClient("")  
            So(err == nil, ShouldEqual, tt.wantErr)  
            redLock, _ := NewLock("LockTimed2")  
            _, err = redLock.lockTimed(tt.args.ctx, tt.args.ttl, tt.args.timeout)  
            So(err == nil, ShouldEqual, tt.wantErr)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     3 * time.Second,  
                    timeout: 2 * time.Second,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            client = nil  
            err := initClient(BOERedisPSM)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            redLock, _ := NewLock("lockTimed")  
            got, err := redLock.lockTimed(tt.args.ctx, tt.args.ttl, tt.args.timeout)  
  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case3", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     2 * time.Second,  
                    timeout: 5 * time.Second,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            client = nil  
            err := initClient(BOERedisPSM)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            redLock, _ := NewLock("LockTimed3")  
            got, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
  
            redLock2, _ := NewLock("LockTimed3")  
            got2, err := redLock2.lockTimed(tt.args.ctx, tt.args.ttl, tt.args.timeout)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got2, ShouldEqual, tt.want)  
        })  
    })  
}  
  
func TestRedLock_LockTimedReliable(t *testing.T) {  
    type args struct {  
        ctx     context.Context  
        ttl     time.Duration  
        timeout time.Duration  
        round   int  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
  
        PatchConvey("test case3", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     30 * time.Second,  
                    timeout: 20 * time.Second,  
                    round:   0,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            client = nil  
            err := initClient("")  
            So(err == nil, ShouldEqual, tt.wantErr)  
  
            redLock, _ := NewLock("LockTimed2")  
            _, err = redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
            So(err == nil, ShouldEqual, tt.wantErr)  
        })  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     30 * time.Second,  
                    timeout: 20 * time.Second,  
                    round:   0,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            err := initClient(BOERedisPSM)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            redLock, _ := NewLock("lockTimedReliable")  
            got, err := redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:     context.Background(),  
                    ttl:     5 * time.Second,  
                    timeout: 2 * time.Second,  
                    round:   0,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            err := initClient(BOERedisPSM)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            redLock, _ := NewLock("LockTimedReliable2")  
            got, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(got, ShouldEqual, tt.want)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            got2, err := redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.timeout, tt.args.round)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(!got2, ShouldEqual, tt.want)  
            _ = redLock.UnLock(tt.args.ctx)  
        })  
    })  
}  
  
func TestRedLock_LockRound(t *testing.T) {  
    type args struct {  
        ctx   context.Context  
        ttl   time.Duration  
        round int  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        err := initClient(BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:   context.Background(),  
                    ttl:   30 * time.Second,  
                    round: 10,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("LockInRound")  
            got, err := redLock.lockRound(tt.args.ctx, tt.args.ttl, tt.args.round, ZeroDuration)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:   context.Background(),  
                    ttl:   30 * time.Second,  
                    round: 10,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("LockInRound2")  
            got, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            got2, err := redLock.lockRound(tt.args.ctx, tt.args.ttl, tt.args.round, ZeroDuration)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
            So(!got2, ShouldEqual, tt.want)  
            _ = redLock.UnLock(tt.args.ctx)  
        })  
        PatchConvey("test case3", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:   context.Background(),  
                    ttl:   30 * time.Second,  
                    round: 2,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock1, _ := NewLock("LockInRound3")  
            got, err := redLock1.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            redLock2, _ := NewLock("LockInRound3")  
            time0 := time.Now()  
            got2, err := redLock2.lockRound(tt.args.ctx, tt.args.ttl, tt.args.round, 2*time.Second)  
            time1 := time.Now()  
            So(time1.Sub(time0) > 4*time.Second, ShouldBeTrue)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
            So(!got2, ShouldEqual, tt.want)  
            _ = redLock1.UnLock(tt.args.ctx)  
        })  
        PatchConvey("test case4", func(c C) {  
            tt := testConfig{  
                args: args{  
                    ctx:   context.Background(),  
                    ttl:   30 * time.Second,  
                    round: 3,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            go func() {  
                redLock1, _ := NewLock("LockInRound4")  
                got, err := redLock1.TryLock(tt.args.ctx, tt.args.ttl)  
                c.So(err != nil, ShouldEqual, tt.wantErr)  
                c.So(got, ShouldEqual, tt.want)  
                time.Sleep(3 * time.Second)  
                _ = redLock1.UnLock(tt.args.ctx)  
                logs.CtxInfo(tt.args.ctx, "1234")  
                logs.Flush()  
            }()  
  
            time.Sleep(200 * time.Millisecond)  
  
            logs.CtxInfo(tt.args.ctx, "5678")  
            logs.Flush()  
  
            redLock2, _ := NewLock("LockInRound4")  
            time0 := time.Now()  
            got2, err := redLock2.lockRound(tt.args.ctx, tt.args.ttl, tt.args.round, 2*time.Second)  
            time1 := time.Now()  
            So(time1.Sub(time0) > 4*time.Second && time1.Sub(time0) < 6*time.Second, ShouldBeTrue)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got2, ShouldEqual, tt.want)  
            _ = redLock2.UnLock(tt.args.ctx)  
        })  
    })  
}  
  
func TestRedLock_LockRoundReliable(t *testing.T) {  
    type args struct {  
        ctx   context.Context  
        ttl   time.Duration  
        time  time.Duration  
        round int  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        err := initClient(BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:   context.Background(),  
                    ttl:   30 * time.Second,  
                    time:  0,  
                    round: 10,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("lockRoundReliable")  
            got, err := redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.time, tt.args.round)  
            defer func() {  
                _ = redLock.UnLock(tt.args.ctx)  
            }()  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:   context.Background(),  
                    ttl:   30 * time.Second,  
                    time:  0,  
                    round: 10,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("LockRoundReliable2")  
            got, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            defer func() {  
                _ = redLock.UnLock(tt.args.ctx)  
            }()  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            got2, err := redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.time, tt.args.round)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
            So(!got2, ShouldEqual, tt.want)  
        })  
        PatchConvey("test error", func() {  
            tt := testConfig{  
                args: args{  
                    ctx:   context.Background(),  
                    ttl:   30 * time.Second,  
                    time:  30 * time.Second,  
                    round: 100,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("LockRoundReliable2")  
            got, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
  
            _, err = redLock.Lock(tt.args.ctx, tt.args.ttl, tt.args.time, tt.args.round)  
            So(err == ErrTimeAndRoundConflict, ShouldBeTrue)  
        })  
    })  
}  
  
func TestRedLock_TryLockReliable(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                },  
                wantErr: false,  
                want:    true,  
            }  
            client = nil  
            err := initClient(BOERedisPSM)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            redLock, _ := NewLock("LockReliable")  
            got, err := redLock.TryLock(tt.args.ctx, 0)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
        })  
    })  
}  
  
func TestRedLock_Renew(t *testing.T) {  
    type args struct {  
        ctx context.Context  
        ttl time.Duration  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        err := initClient(BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                    ttl: 3 * time.Second,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("RenewLock")  
            result, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(result, ShouldEqual, tt.want)  
  
            time.Sleep(time.Second * 2)  
  
            got, err := redLock.Renew(tt.args.ctx, tt.args.ttl)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldEqual, tt.want)  
  
            time.Sleep(time.Second * 2)  
            err = redLock.UnLock(tt.args.ctx)  
            So(err == nil, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                    ttl: 5 * time.Second,  
                },  
                wantErr: true,  
                want:    true,  
            }  
            redLock, _ := NewLock("RenewLock2")  
            redLock2, _ := NewLock("RenewLock2")  
            result, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err == nil, ShouldEqual, tt.wantErr)  
            So(result, ShouldEqual, tt.want)  
            _, err = redLock2.Renew(tt.args.ctx, tt.args.ttl)  
            So(err == ErrNotLockOwner, ShouldEqual, tt.wantErr)  
        })  
        PatchConvey("test case3", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                    ttl: 5 * time.Second,  
                },  
                wantErr: true,  
                want:    true,  
            }  
            redLock, _ := NewLock("RenewLock3")  
            _, err = redLock.Renew(tt.args.ctx, tt.args.ttl)  
            So(err == ErrNotExitLock, ShouldEqual, tt.wantErr)  
        })  
    })  
}  
  
func TestRedLock_Unlock(t *testing.T) {  
    type args struct {  
        ctx context.Context  
        ttl time.Duration  
    }  
    type testConfig struct {  
        args    args  
        want    bool  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        err := initClient(BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                    ttl: 15 * time.Second,  
                },  
                wantErr: false,  
                want:    true,  
            }  
            redLock, _ := NewLock("UnLock")  
            got1, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            err = redLock.UnLock(tt.args.ctx)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got1, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                    ttl: 15 * time.Second,  
                },  
                wantErr: true,  
                want:    true,  
            }  
            redLock, _ := NewLock("Unlock2")  
            redLock2, _ := NewLock("Unlock2")  
            got1, err := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
            So(err == nil, ShouldEqual, tt.wantErr)  
            err = redLock2.UnLock(tt.args.ctx)  
            So(err == ErrNotLockOwner, ShouldEqual, tt.wantErr)  
            So(got1, ShouldEqual, tt.want)  
        })  
        PatchConvey("test case3", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                    ttl: 15 * time.Second,  
                },  
                wantErr: true,  
                want:    false,  
            }  
            redLock, _ := NewLock("Unlock3")  
            err := redLock.UnLock(tt.args.ctx)  
            So(err == ErrNotExitLock, ShouldEqual, tt.wantErr)  
        })  
    })  
}  
  
func TestGetRedLockKey(t *testing.T) {  
    type args struct {  
        dest string  
    }  
    type testConfig struct {  
        args args  
        want string  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    dest: "dest",  
                },  
                want: serPSM + "_" + "dest",  
            }  
            got := RedLockKey(tt.args.dest)  
            So(got, ShouldEqual, tt.want)  
        })  
    })  
}  
  
func Test_getClient(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args    args  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        err := initClient(BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                },  
                wantErr: false,  
            }  
            _, err := getClient(tt.args.ctx)  
            So(err != nil, ShouldEqual, tt.wantErr)  
        })  
    })  
}  
  
func TestRedLock_TryLockConcur(t *testing.T) {  
    type args struct {  
        ctx context.Context  
        ttl time.Duration  
    }  
    type testConfig struct {  
        args args  
        want int  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                    ttl: 30 * time.Second,  
                },  
                want: 1,  
            }  
            err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
            if err != nil {  
                return  
            }  
            counter := 0  
            for i := 0; i < 10000; i++ {  
                go func(idx int) {  
                    redLock, _ := NewLock("TryLockConcur")  
                    got, _ := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
                    if got {  
                        counter++  
                    }  
                }(i)  
            }  
            time.Sleep(time.Second * 5)  
            for i := 0; i < 10000; i++ {  
                go func(idx int) {  
                    redLock, _ := NewLock("TryLockConcur")  
                    got, _ := redLock.TryLock(tt.args.ctx, tt.args.ttl)  
                    if got {  
                        counter++  
                    }  
                }(i)  
            }  
            time.Sleep(time.Second * 5)  
            logs.Info("[BLock/TestRedLock_TryLock] count=%v", counter)  
            logs.Flush()  
            So(counter, ShouldEqual, tt.want)  
        })  
    })  
}  
  
func TestRedLock_TryLockReliableConcur(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args args  
        want int  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                },  
                want: 1,  
            }  
            err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
            if err != nil {  
                return  
            }  
            counter := 0  
            for i := 0; i < 100; i++ {  
                go func() {  
                    redLock, _ := NewLock("TryLockReliableConcur")  
                    got, _ := redLock.TryLock(tt.args.ctx, 0)  
                    if got {  
                        counter++  
                    }  
                }()  
            }  
            time.Sleep(time.Second * 30)  
            for i := 0; i < 100; i++ {  
                go func() {  
                    redLock, _ := NewLock("TryLockReliableConcur")  
                    got, _ := redLock.TryLock(tt.args.ctx, 0)  
                    if got {  
                        counter++  
                    }  
                }()  
            }  
            So(counter, ShouldEqual, tt.want)  
        })  
    })  
}  
  
func TestRedLock_WatchDog(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args args  
        want int  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                },  
                want: 1,  
            }  
            err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
            if err != nil {  
                return  
            }  
            var ops uint64 = 0  
            for i := 0; i < 200; i++ {  
                go func(index int) {  
                    redLock, _ := NewLock("TryLockWatchDog" + strconv.Itoa(index))  
                    got, _ := redLock.TryLock(tt.args.ctx, 0)  
                    logs.CtxInfo(tt.args.ctx, "[BLock/TestRedLock_WatchDog]:  got=%v,index=%v \n", got, index)  
                    time.Sleep(time.Second * 30)  
                    if got {  
                        atomic.AddUint64(&ops, 1)  
                        err = redLock.UnLock(tt.args.ctx)  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestRedLock_WatchDog] unlock=%v index=%v \n", err == nil, index)  
                    }  
                }(i)  
            }  
            time.Sleep(time.Second * 40)  
            logs.CtxInfo(tt.args.ctx, "lockSuccess=%v \n", ops)  
        })  
        PatchConvey("test case2", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                },  
                want: 1,  
            }  
            err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
            if err != nil {  
                return  
            }  
        })  
    })  
}  
  
func TestRedLock_WatchDogCost(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args args  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            tt := testConfig{  
                args: args{  
                    ctx: context.Background(),  
                },  
            }  
            err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
            if err != nil {  
                return  
            }  
            var ops uint64 = 0  
            var totalCost uint64 = 0  
            for i := 0; i < 200; i++ {  
                go func(index int) {  
                    redLock, _ := NewLock("TryLockWatchDog" + strconv.Itoa(index))  
                    begin := time.Now()  
                    got, _ := redLock.TryLock(tt.args.ctx, 0)  
                    if got {  
                        atomic.AddUint64(&ops, 1)  
                        err = redLock.UnLock(tt.args.ctx)  
                        if err != nil {  
                            t.Errorf("TestRedLock_WatchDogCost unlock error")  
                        }  
                        cost := time.Since(begin).Milliseconds()  
                        atomic.AddUint64(&totalCost, uint64(cost))  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestRedLock_WatchDog] unlock=%v index=%v cost=%v \n", err == nil, index, cost)  
                    }  
                }(i)  
            }  
            time.Sleep(time.Second * 10)  
            logs.CtxInfo(tt.args.ctx, "lockSuccess=%v, cost=%v, avgCost=%v \n", ops, totalCost, totalCost/ops)  
            logs.Flush()  
        })  
    })  
}  
  
func TestGracefulUnlock(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args args  
        want int  
    }  
  
    PatchConvey("graceful unlock", t, func() {  
        tt := testConfig{  
            args: args{  
                ctx: context.Background(),  
            },  
            want: 100,  
        }  
        err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("only long lease lock graceful unlock", func() {  
            var ops uint64 = 0  
            goNum := tt.want  
            var wg sync.WaitGroup  
            wg.Add(goNum)  
            for i := 0; i < goNum; i++ {  
                go func(index int) {  
                    defer wg.Done()  
  
                    redLock, _ := NewLock("LongLeaseLockDuration" + strconv.Itoa(index))  
                    got, _ := redLock.TryLock(tt.args.ctx, 60000*time.Millisecond)  
                    if got {  
                        atomic.AddUint64(&ops, 1)  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] lock Success lock=%v \n", redLock.redisKey)  
                    } else {  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] lock fail lock=%v \n", redLock.redisKey)  
                    }  
                }(i)  
            }  
            wg.Wait()  
            logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] ops=%v \n", ops)  
  
            GracefulUnlock(tt.args.ctx)  
            logs.Flush()  
        })  
        PatchConvey("only reliable lock graceful unlock", func() {  
            var ops uint64 = 0  
            goNum := tt.want  
            var wg sync.WaitGroup  
            wg.Add(goNum)  
            for i := 0; i < goNum; i++ {  
                go func(index int) {  
                    defer wg.Done()  
  
                    redLock, _ := NewLock("ReliableLock" + strconv.Itoa(index))  
                    got, _ := redLock.TryLock(tt.args.ctx, 0)  
                    if got {  
                        atomic.AddUint64(&ops, 1)  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] lock Success lock=%v \n", redLock.redisKey)  
                    } else {  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] lock fail lock=%v \n", redLock.redisKey)  
                    }  
                }(i)  
            }  
            wg.Wait()  
            logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] ops=%v \n", ops)  
  
            GracefulUnlock(tt.args.ctx)  
            logs.Flush()  
        })  
        PatchConvey("graceful unlock long lease lock and reliable lock", func() {  
            var longLeaseLockOps uint64 = 0  
            var reliableLockOps uint64 = 0  
            goNum := tt.want  
            var wg sync.WaitGroup  
            wg.Add(goNum)  
            for i := 0; i < goNum; i++ {  
                go func(index int) {  
                    defer wg.Done()  
  
                    redLongLeaseLock, _ := NewLock("LongLeaseLock2" + strconv.Itoa(index))  
                    got1, _ := redLongLeaseLock.TryLock(tt.args.ctx, 60000*time.Millisecond)  
                    if got1 {  
                        atomic.AddUint64(&longLeaseLockOps, 1)  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] long lease lock Success lock=%v \n", redLongLeaseLock.redisKey)  
                    } else {  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] long lease lock fail lock=%v \n", redLongLeaseLock.redisKey)  
                    }  
  
                    redReliableLock, _ := NewLock("ReliableLock2" + strconv.Itoa(index))  
                    got2, _ := redReliableLock.TryLock(tt.args.ctx, -1)  
                    if got2 {  
                        atomic.AddUint64(&reliableLockOps, 1)  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] reliable lock Success lock=%v \n", redReliableLock.redisKey)  
                    } else {  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] reliable lock fail lock=%v \n", redReliableLock.redisKey)  
                    }  
                }(i)  
            }  
            wg.Wait()  
  
            logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] before graceful unlock longLeaseLockMap len=%v, watchDogMap len=%v \n", longLeaseLockMap.Count(), watchDogMap.Count())  
            GracefulUnlock(tt.args.ctx)  
  
            logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] after graceful unlock longLeaseLockMap len=%v, watchDogMap len=%v \n", longLeaseLockMap.Count(), watchDogMap.Count())  
  
            for i := 0; i < 10; i++ {  
                redLongLeaseLock, _ := NewLock("LongLeaseLock2" + strconv.Itoa(i))  
                got1, _ := redLongLeaseLock.TryLock(tt.args.ctx, 10*time.Second)  
                So(got1, ShouldEqual, true)  
                err := redLongLeaseLock.UnLock(tt.args.ctx)  
                So(err == nil, ShouldEqual, true)  
                redReliableLock, _ := NewLock("ReliableLock2" + strconv.Itoa(i))  
                got2, _ := redReliableLock.TryLock(tt.args.ctx, -1)  
                So(got2, ShouldEqual, true)  
                err = redReliableLock.UnLock(tt.args.ctx)  
                So(err == nil, ShouldEqual, true)  
            }  
            logs.Flush()  
        })  
    })  
}  
  
func TestGracefulUnlockCronJob(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args args  
    }  
    PatchConvey("test", t, func() {  
        tt := testConfig{  
            args: args{  
                ctx: context.Background(),  
            },  
        }  
        err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test Cron job", func() {  
            redLock1, _ := NewLock("LongLeaseLockCronJob1")  
            got1, _ := redLock1.TryLock(tt.args.ctx, 40*time.Second)  
            So(got1, ShouldEqual, true)  
  
            redLock2, _ := NewLock("LongLeaseLockCronJob2")  
            got2, _ := redLock2.TryLock(tt.args.ctx, 50*time.Second)  
            So(got2, ShouldEqual, true)  
  
            redLock3, _ := NewLock("LongLeaseLockCronJob3")  
            got3, _ := redLock3.TryLock(tt.args.ctx, 70*time.Second)  
            So(got3, ShouldEqual, true)  
  
            redLock4, _ := NewLock("LongLeaseLockCronJob4")  
            got4, _ := redLock4.TryLock(tt.args.ctx, 80*time.Second)  
            So(got4, ShouldEqual, true)  
  
            So(longLeaseLockMap.Count() == 4, ShouldEqual, true)  
  
            time.Sleep(time.Second * 65)  
            So(longLeaseLockMap.Count() == 2, ShouldEqual, true)  
  
            GracefulUnlock(tt.args.ctx)  
            err := redLock3.UnLock(tt.args.ctx)  
            So(err == ErrNotExitLock, ShouldEqual, true)  
            err = redLock4.UnLock(tt.args.ctx)  
            So(err == ErrNotExitLock, ShouldEqual, true)  
  
            logs.Flush()  
        })  
    })  
}  
  
func TestCron(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args args  
    }  
    PatchConvey("test", t, func() {  
        tt := testConfig{  
            args: args{  
                ctx: context.Background(),  
            },  
        }  
        err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
        if err != nil {  
            return  
        }  
        PatchConvey("test Cron job", func() {  
            time.Sleep(time.Second * 20)  
            var longLeaseLockOps uint64 = 0  
            var reliableLockOps uint64 = 0  
            goNum := 20  
            var wg sync.WaitGroup  
            wg.Add(goNum)  
            for i := 0; i < goNum; i++ {  
                go func(index int) {  
                    defer wg.Done()  
  
                    redLongLeaseLock, _ := NewLock("CronLongLeaseLock" + strconv.Itoa(index))  
                    longLeaseLockDuration := 30 * time.Second  
                    if index >= goNum/2 {  
                        longLeaseLockDuration = 70 * time.Second  
                    }  
                    got1, _ := redLongLeaseLock.TryLock(tt.args.ctx, longLeaseLockDuration)  
                    if got1 {  
                        atomic.AddUint64(&longLeaseLockOps, 1)  
                    } else {  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] long lease lock fail lock=%v \n", redLongLeaseLock.redisKey)  
                    }  
  
                    redReliableLock, _ := NewLock("CronReliableLock" + strconv.Itoa(index))  
                    got2, _ := redReliableLock.TryLock(tt.args.ctx, -1)  
                    if got2 {  
                        atomic.AddUint64(&reliableLockOps, 1)  
                    } else {  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestGracefulUnlock] reliable lock fail lock=%v \n", redReliableLock.redisKey)  
                    }  
                }(i)  
            }  
            wg.Wait()  
  
            So(longLeaseLockMap.Count(), ShouldEqual, goNum)  
            So(watchDogMap.Count(), ShouldEqual, goNum)  
  
            time.Sleep(60 * time.Second)  
  
            So(longLeaseLockMap.Count(), ShouldEqual, goNum/2)  
            So(watchDogMap.Count(), ShouldEqual, goNum)  
  
            GracefulUnlock(tt.args.ctx)  
  
            for i := 0; i < 10; i++ {  
                redLongLeaseLock, _ := NewLock("CronLongLeaseLock" + strconv.Itoa(i))  
                got1, _ := redLongLeaseLock.TryLock(tt.args.ctx, 10*time.Second)  
                So(got1, ShouldEqual, true)  
                err := redLongLeaseLock.UnLock(tt.args.ctx)  
                So(err == nil, ShouldEqual, true)  
                redReliableLock, _ := NewLock("CronReliableLock" + strconv.Itoa(i))  
                got2, _ := redReliableLock.TryLock(tt.args.ctx, -1)  
                So(got2, ShouldEqual, true)  
                err = redReliableLock.UnLock(tt.args.ctx)  
                So(err == nil, ShouldEqual, true)  
            }  
            logs.Flush()  
        })  
    })  
}  
  
func TestTimeout(t *testing.T) {  
    type args struct {  
        ctx context.Context  
    }  
    type testConfig struct {  
        args args  
    }  
    PatchConvey("test", t, func() {  
        tt := testConfig{  
            args: args{  
                ctx: context.Background(),  
            },  
        }  
        err := Init(tt.args.ctx, "BLockTest", BOERedisPSM)  
        if err != nil {  
            return  
        }  
  
        PatchConvey("try lock fail then retry", func() {  
            redLock, _ := NewLock("retry1")  
            client, err := getClient(tt.args.ctx)  
            So(err == nil, ShouldEqual, true)  
            got, err := redLock.retryLock(client, 5*time.Second)  
            So(got, ShouldEqual, true)  
        })  
        PatchConvey("try lock success but timeout then retry", func() {  
            redLock, _ := NewLock("retry2")  
            got, err := redLock.TryLock(tt.args.ctx, 5*time.Second)  
            So(got, ShouldEqual, true)  
  
            client, err := getClient(tt.args.ctx)  
            So(err == nil, ShouldEqual, true)  
  
            got2, err := redLock.retryLock(client, 5*time.Second)  
            So(got2, ShouldEqual, true)  
        })  
        PatchConvey("retry fail", func() {  
            redLock1, _ := NewLock("retry3")  
            redLock2, _ := NewLock("retry3")  
            got, err := redLock1.TryLock(tt.args.ctx, 5*time.Second)  
            So(got, ShouldEqual, true)  
  
            client, err := getClient(tt.args.ctx)  
            So(err == nil, ShouldEqual, true)  
  
            got2, err := redLock2.retryLock(client, 5*time.Second)  
            So(got2, ShouldEqual, false)  
            So(err == nil, ShouldEqual, true)  
        })  
        PatchConvey("try lock batch has timeout", func() {  
            count := 300  
            var wg sync.WaitGroup  
            wg.Add(count)  
            for i := 0; i < count; i++ {  
                go func(index int) {  
                    defer wg.Done()  
                    redLock, _ := NewLock("timeout" + strconv.Itoa(index))  
                    got, err := redLock.TryLock(tt.args.ctx, 2*time.Millisecond)  
                    if !got {  
                        logs.CtxInfo(tt.args.ctx, "[BLock/TestRedLock_WatchDog] lock fail index=%v \n", index)  
                    }  
                    if err != nil {  
                        logs.CtxError(tt.args.ctx, "err = %v", err)  
                    }  
                }(i)  
            }  
            wg.Wait()  
            logs.Flush()  
        })  
    })  
}  
  
func TestMarshal(t *testing.T) {  
    type args struct {  
        redLock *RedLock  
        ctx     context.Context  
    }  
    type testConfig struct {  
        args    args  
        want    []byte  
        wantErr bool  
    }  
    PatchConvey("test", t, func() {  
        _ = Init(context.Background(), "BLockTest", BOERedisPSM)  
        PatchConvey("test case1", func() {  
            redLock0, _ := NewLock("MarshalLock1")  
            tt := testConfig{  
                args: args{  
                    redLock: &redLock0,  
                    ctx:     context.Background(),  
                },  
                wantErr: false,  
            }  
  
            got, err := redLock0.TryLock(tt.args.ctx, 20*time.Second)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldBeTrue)  
  
            got1, err := Marshal(tt.args.redLock)  
            So(err != nil, ShouldEqual, tt.wantErr)  
  
            redLock1, err := Unmarshal(got1)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            err = redLock1.UnLock(tt.args.ctx)  
            So(err != nil, ShouldEqual, tt.wantErr)  
        })  
        PatchConvey("test case2", func() {  
            redLock0, _ := NewLock("MarshalLock2")  
            tt := testConfig{  
                args: args{  
                    redLock: &redLock0,  
                    ctx:     context.Background(),  
                },  
                wantErr: false,  
            }  
            got, err := redLock0.TryLock(tt.args.ctx, 40*time.Second)  
            So(err != nil, ShouldEqual, tt.wantErr)  
            So(got, ShouldBeTrue)  
            _, err = Marshal(tt.args.redLock)  
            So(errors.Is(err, ErrNoMarshalLock), ShouldBeTrue)  
            _ = redLock0.UnLock(tt.args.ctx)  
        })  
        PatchConvey("test case3", func() {  
            Mock(json.Marshal).Return(nil, ErrNoMarshalLock).Build()  
            redLock3, _ := NewLock("MarshalLock3")  
            tt := testConfig{  
                args: args{  
                    redLock: &redLock3,  
                },  
                wantErr: false,  
            }  
            _, err := Marshal(tt.args.redLock)  
            So(err != nil, ShouldBeTrue)  
        })  
    })  
}  
  
func TestRedLock_LockTarget(t *testing.T) {  
    type fields struct {  
        target        string  
        redisKey      string  
        redisValue    string  
        reliable      bool  
        longLeaseLock bool  
        timeToLeave   time.Time  
    }  
    type testConfig struct {  
        fields fields  
        want   string  
    }  
    PatchConvey("test", t, func() {  
        //your mock code...  
        PatchConvey("test case1", func() {  
            err := Init(context.Background(), "BLockTest", BOERedisPSM)  
            lock, err := NewLock("LockTarget")  
            So(err == nil, ShouldBeTrue)  
            got := lock.LockTarget()  
            So(got == "LockTarget", ShouldBeTrue)  
        })  
    })  
}  
  
func TestRedLockKey(t *testing.T) {  
    type args struct {  
        dest string  
    }  
    type testConfig struct {  
        args args  
        want string  
    }  
    PatchConvey("test", t, func() {  
        //your mock code...  
        PatchConvey("test case1", func() {  
            err := Init(context.Background(), "BLockTest", BOERedisPSM)  
            lock, err := NewLock("LockKey")  
            So(err == nil, ShouldBeTrue)  
            got := lock.LockKey()  
            So(got == "BLockTest_LockKey", ShouldBeTrue)  
        })  
    })  
}  
  
func TestRedLock_LockValue(t *testing.T) {  
    type fields struct {  
        target        string  
        redisKey      string  
        redisValue    string  
        reliable      bool  
        longLeaseLock bool  
        timeToLeave   time.Time  
    }  
    type testConfig struct {  
        fields fields  
        want   string  
    }  
    PatchConvey("test", t, func() {  
        PatchConvey("test case1", func() {  
            err := Init(context.Background(), "BLockTest", BOERedisPSM)  
            lock, err := NewLock("LockValue")  
            So(err == nil, ShouldBeTrue)  
            got := lock.LockValue()  
            So(got == lock.redisValue, ShouldBeTrue)  
        })  
    })  
}  
```

---
# 5 引用