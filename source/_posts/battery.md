-----
title: Android 耗电量统计
tags: 
   - Android
   - Battery
-----

 将电池比喻为蓄水池，则系统统计的（DrainType）就是不同类型的排水。

## wakeLock 耗电计算


```
 public WakelockPowerCalculator(PowerProfile profile) {
        mPowerWakelock = profile.getAveragePower(PowerProfile.POWER_CPU_AWAKE);
    }


    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
        long wakeLockTimeUs = 0;
        final ArrayMap<String, ? extends BatteryStats.Uid.Wakelock> wakelockStats =
                u.getWakelockStats();
        final int wakelockStatsCount = wakelockStats.size();
        for (int i = 0; i < wakelockStatsCount; i++) {
            final BatteryStats.Uid.Wakelock wakelock = wakelockStats.valueAt(i);

            // Only care about partial wake locks since full wake locks
            // are canceled when the user turns the screen off.
            BatteryStats.Timer timer = wakelock.getWakeTime(BatteryStats.WAKE_TYPE_PARTIAL);
            if (timer != null) {
                wakeLockTimeUs += timer.getTotalTimeLocked(rawRealtimeUs, statsType);
            }
        }
        app.wakeLockTimeMs = wakeLockTimeUs / 1000; // convert to millis
        mTotalAppWakelockTimeMs += app.wakeLockTimeMs;

        // Add cost of holding a wake lock.
        app.wakeLockPowerMah = (app.wakeLockTimeMs * mPowerWakelock) / (1000*60*60);
        if (DEBUG && app.wakeLockPowerMah != 0) {
            Log.d(TAG, "UID " + u.getUid() + ": wake " + app.wakeLockTimeMs
                    + " power=" + BatteryStatsHelper.makemAh(app.wakeLockPowerMah));
        }
    }

```


wakelock 阻止cpu进入休眠，所以使用了cpu_awake的值进行计算。
首先获得Wakelock的数量，然后逐个遍历得到每个Wakelock对象，得到该对象后，得到BatteryStats.WAKE_TYPE_PARTIAL的唤醒时间，然后累加，其实wakelock有4种，为什么只取partial的时间,具体代码google也没解释的很清楚,只是用一句注释打发了我们。得到总时间后，就可以与构造方法中的单位时间waklock消耗电量相乘得到Wakelock消耗的总电量。



Q:为啥app下有wifi耗电，还要单独一个DrainType统计wifi耗电，是否是重复统计？所有的app和硬件耗电相加为啥是100%？
A:系统会先计算app耗电，将归属在对应uid下面的耗电先计算出来，后再将剩余电量计算出来并归属于对应硬件。以wifi为例：
```
 public WifiPowerCalculator(PowerProfile profile) {
        mIdleCurrentMa = profile.getAveragePower(PowerProfile.POWER_WIFI_CONTROLLER_IDLE);
        mTxCurrentMa = profile.getAveragePower(PowerProfile.POWER_WIFI_CONTROLLER_TX);
        mRxCurrentMa = profile.getAveragePower(PowerProfile.POWER_WIFI_CONTROLLER_RX);
    }

    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
        //获取对应uid下的counter
        final BatteryStats.ControllerActivityCounter counter = u.getWifiControllerActivity();
        if (counter == null) {
            return;
        }
		//对应uid下的time
        final long idleTime = counter.getIdleTimeCounter().getCountLocked(statsType);
        final long txTime = counter.getTxTimeCounters()[0].getCountLocked(statsType);
        final long rxTime = counter.getRxTimeCounter().getCountLocked(statsType);
        app.wifiRunningTimeMs = idleTime + rxTime + txTime;
		//已经计算的应用的wifi总耗时
        mTotalAppRunningTime += app.wifiRunningTimeMs;

		//计算对应uid的app耗电
        app.wifiPowerMah =
                ((idleTime * mIdleCurrentMa) + (txTime * mTxCurrentMa) + (rxTime * mRxCurrentMa))
                / (1000*60*60);
		//已经计算的应用的wifi总耗电
        mTotalAppPowerDrain += app.wifiPowerMah;

        app.wifiRxPackets = u.getNetworkActivityPackets(BatteryStats.NETWORK_WIFI_RX_DATA,
                statsType);
        app.wifiTxPackets = u.getNetworkActivityPackets(BatteryStats.NETWORK_WIFI_TX_DATA,
                statsType);
        app.wifiRxBytes = u.getNetworkActivityBytes(BatteryStats.NETWORK_WIFI_RX_DATA,
                statsType);
        app.wifiTxBytes = u.getNetworkActivityBytes(BatteryStats.NETWORK_WIFI_TX_DATA,
                statsType);

        if (DEBUG && app.wifiPowerMah != 0) {
            Log.d(TAG, "UID " + u.getUid() + ": idle=" + idleTime + "ms rx=" + rxTime + "ms tx=" +
                    txTime + "ms power=" + BatteryStatsHelper.makemAh(app.wifiPowerMah));
        }
    }

	//这个方法是在所有app耗电计算完成后调用的
    @Override
    public void calculateRemaining(BatterySipper app, BatteryStats stats, long rawRealtimeUs,
                                   long rawUptimeUs, int statsType) {
		//获取总counter
        final BatteryStats.ControllerActivityCounter counter = stats.getWifiControllerActivity();

        final long idleTimeMs = counter.getIdleTimeCounter().getCountLocked(statsType);
        final long txTimeMs = counter.getTxTimeCounters()[0].getCountLocked(statsType);
        final long rxTimeMs = counter.getRxTimeCounter().getCountLocked(statsType);
		//计算剩余的time
        app.wifiRunningTimeMs = Math.max(0,
                (idleTimeMs + rxTimeMs + txTimeMs) - mTotalAppRunningTime);

        double powerDrainMah = counter.getPowerCounter().getCountLocked(statsType)
                / (double)(1000*60*60);
        if (powerDrainMah == 0) {
            // Some controllers do not report power drain, so we can calculate it here.
            powerDrainMah = ((idleTimeMs * mIdleCurrentMa) + (txTimeMs * mTxCurrentMa)
                    + (rxTimeMs * mRxCurrentMa)) / (1000*60*60);
        }
		//计算剩余的耗电
        app.wifiPowerMah = Math.max(0, powerDrainMah - mTotalAppPowerDrain);

        if (DEBUG) {
            Log.d(TAG, "left over WiFi power: " + BatteryStatsHelper.makemAh(app.wifiPowerMah));
        }
    }
```

## IDLE计算
不知道IDLE计算啥原理
```
        mTypeBatteryUptimeUs = mStats.computeBatteryUptime(rawUptimeUs, mStatsType);
        mTypeBatteryRealtimeUs = mStats.computeBatteryRealtime(rawRealtimeUs, mStatsType)

        final double suspendPowerMaMs = (mTypeBatteryRealtimeUs / 1000) *
                mPowerProfile.getAveragePower(PowerProfile.POWER_CPU_IDLE);
        final double idlePowerMaMs = (mTypeBatteryUptimeUs / 1000) *
                mPowerProfile.getAveragePower(PowerProfile.POWER_CPU_AWAKE);
        final double totalPowerMah = (suspendPowerMaMs + idlePowerMaMs) / (60 * 60 * 1000);
```

```
/**
     * Returns the total, last, or current battery uptime in microseconds.
     * 上次或当前电池正常运行时间。
     * @param curTime the elapsed realtime in microseconds.
     * @param which one of STATS_SINCE_CHARGED, STATS_SINCE_UNPLUGGED, or STATS_CURRENT.
     */
    public abstract long computeBatteryUptime(long curTime, int which);

    /**
     * Returns the total, last, or current battery realtime in microseconds.
     * 最后或当前电池实时时间
     * @param curTime the current elapsed realtime in microseconds.
     * @param which one of STATS_SINCE_CHARGED, STATS_SINCE_UNPLUGGED, or STATS_CURRENT.
     */
    public abstract long computeBatteryRealtime(long curTime, int which);
```

## CPU 计算
CPU 部分没有remaining
```
public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
        app.cpuTimeMs = (u.getUserCpuTimeUs(statsType) + u.getSystemCpuTimeUs(statsType)) / 1000;

        // Aggregate total time spent on each cluster.
        long totalTime = 0;
        final int numClusters = mProfile.getNumCpuClusters();
        for (int cluster = 0; cluster < numClusters; cluster++) {
            final int speedsForCluster = mProfile.getNumSpeedStepsInCpuCluster(cluster);
            for (int speed = 0; speed < speedsForCluster; speed++) {
                totalTime += u.getTimeAtCpuSpeed(cluster, speed, statsType);
            }
        }
        totalTime = Math.max(totalTime, 1);

        double cpuPowerMaMs = 0;
        for (int cluster = 0; cluster < numClusters; cluster++) {
            final int speedsForCluster = mProfile.getNumSpeedStepsInCpuCluster(cluster);
            for (int speed = 0; speed < speedsForCluster; speed++) {
				//获取对应cluster，对应speed下所使用时间的比例
                final double ratio = (double) u.getTimeAtCpuSpeed(cluster, speed, statsType) /
                        totalTime;
                final double cpuSpeedStepPower = ratio * app.cpuTimeMs *
                        mProfile.getAveragePowerForCpu(cluster, speed);
                if (DEBUG && ratio != 0) {
                    Log.d(TAG, "UID " + u.getUid() + ": CPU cluster #" + cluster + " step #"
                            + speed + " ratio=" + BatteryStatsHelper.makemAh(ratio) + " power="
                            + BatteryStatsHelper.makemAh(cpuSpeedStepPower / (60 * 60 * 1000)));
                }
                cpuPowerMaMs += cpuSpeedStepPower;
            }
        }
        app.cpuPowerMah = cpuPowerMaMs / (60 * 60 * 1000);
```

## OVERCOUNTED，UNACCOUNTED 计算

```
mMinDrainedPower = (mStats.getLowDischargeAmountSinceCharge()
        * mPowerProfile.getBatteryCapacity()) / 100;
mMaxDrainedPower = (mStats.getHighDischargeAmountSinceCharge()
        * mPowerProfile.getBatteryCapacity()) / 100;
 
mTotalPower = mComputedPower;
if (mStats.getLowDischargeAmountSinceCharge() > 1) {
    if (mMinDrainedPower > mComputedPower) {
        double amount = mMinDrainedPower - mComputedPower;
        mTotalPower = mMinDrainedPower;
        BatterySipper bs = new BatterySipper(DrainType.UNACCOUNTED, null, amount);
 
         
    } else if (mMaxDrainedPower < mComputedPower) {
        double amount = mComputedPower - mMaxDrainedPower;
 
        // Insert the BatterySipper in its sorted position.
        BatterySipper bs = new BatterySipper(DrainType.OVERCOUNTED, null, amount);
         
    }
}
//其中getLowDischargeAmountSinceCharge
@Override
public int getLowDischargeAmountSinceCharge() {
    synchronized(this) {
        int val = mLowDischargeAmountSinceCharge;
        if (mOnBattery && mDischargeCurrentLevel < mDischargeUnplugLevel) {
            val += mDischargeUnplugLevel-mDischargeCurrentLevel-1;
        }
        return val;
    }
}
//mLowDischargeAmountSinceCharge在setOnBatteryLocked 会改变这个值。mLowDischargeAmountSinceCharge 和mHighDischargeAmountSinceCharge 这两个变量中，这两个变量差1，是一个误差，受电池level精度影响
void setOnBatteryLocked(final long mSecRealtime, final long mSecUptime, final boolean onBattery,
        final int oldStatus, final int level, final int chargeUAh) {
.......
if (onBattery) {
....
} else {
    mLastChargingStateLevel = level;
    mOnBattery = mOnBatteryInternal = false;
    pullPendingStateUpdatesLocked();
    mHistoryCur.batteryLevel = (byte)level;
    mHistoryCur.states |= HistoryItem.STATE_BATTERY_PLUGGED_FLAG;
    if (DEBUG_HISTORY) Slog.v(TAG, "Battery plugged to: "
            + Integer.toHexString(mHistoryCur.states));
    addHistoryRecordLocked(mSecRealtime, mSecUptime);
    mDischargeCurrentLevel = mDischargePlugLevel = level;
    if (level < mDischargeUnplugLevel) {//level代表当前电量，mDischargeUnplugLevel代表上一次拔去usb线的电量 
        mLowDischargeAmountSinceCharge += mDischargeUnplugLevel-level-1;//累计消耗的电量 
        mHighDischargeAmountSinceCharge += mDischargeUnplugLevel-level;
    }
    updateDischargeScreenLevelsLocked(screenOn, screenOn);
    updateTimeBasesLocked(false, !screenOn, uptime, realtime);
    mChargeStepTracker.init();
    mLastChargeStepLevel = level;
    mMaxChargeStepLevel = level;
    mInitStepMode = mCurStepMode;
    mModStepMode = 0;
}
//再看下getLowDischargeAmountSinceCharge
@Override
public int getLowDischargeAmountSinceCharge() {
    synchronized(this) {
        int val = mLowDischargeAmountSinceCharge;
        //表示现在正在用电状态，mDischargeCurrentLevel 变量代表用电的时候的电量，会时时更新，mDischargeUnplugLevel代表上一次拔去usb线的一个电量
        if (mOnBattery && mDischargeCurrentLevel < mDischargeUnplugLevel) {
            val += mDischargeUnplugLevel-mDischargeCurrentLevel-1;//也就是加上最近的一次消耗电量 
        }
        return val;
    }
}
//再看下UNACCOUNTED，OVERCOUNTED计算
mTotalPower = mComputedPower;
if (mStats.getLowDischargeAmountSinceCharge() > 1) {//只统计消耗的电量大于1
    if (mMinDrainedPower > mComputedPower) {//如果实际总的消耗电量(min)比统计的电量大
        double amount = mMinDrainedPower - mComputedPower;
        mTotalPower = mMinDrainedPower;
        BatterySipper bs = new BatterySipper(DrainType.UNACCOUNTED, null, amount);
 
         
    } else if (mMaxDrainedPower < mComputedPower) {//如果实际总的消耗电量(max)比统计的电量小
        double amount = mComputedPower - mMaxDrainedPower;
 
        // Insert the BatterySipper in its sorted position.
        BatterySipper bs = new BatterySipper(DrainType.OVERCOUNTED, null, amount);
         
    }
}
```