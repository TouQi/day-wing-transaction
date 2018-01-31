# 日内回转交易(股票)
基于股票日内偏离度回归的日内回转策略

## 源码
```python
# coding=utf-8
from __future__ import print_function, absolute_import, unicode_literals
try:
    import talib
except:
    print('请安装TA-Lib库')
from gm.api import *
'''
本策略首先买入SHSE.600000股票10000股
随后根据60s的数据来计算MACD(12,26,9)线,并在MACD>0的时候买入100股,MACD<0的时候卖出100股
但每日操作的股票数不超过原有仓位,并于收盘前把仓位调整至开盘前的仓位
回测数据为:SHSE.600000的60s数据
回测时间为:2017-09-01 08:00:00到2017-10-01 16:00:00
'''
def init(context):
    # 设置标的股票
    context.symbol = 'SHSE.600000'
    # 用于判定第一个仓位是否成功开仓
    context.first = 0
    # 订阅浦发银行, bar频率为1min
    subscribe(symbols=context.symbol, frequency='60s', count=35)
    # 日内回转每次交易100股
    context.trade_n = 100
    # 获取昨今天的时间
    context.day = [0, 0]
    # 用于判断是否触发了回转逻辑的计时
    context.ending = 0
def on_bar(context, bars):
    bar = bars[0]
    if context.first == 0:
        # 最开始配置仓位
        # 需要保持的总仓位
        context.total = 10000
        # 购买10000股浦发银行股票
        order_volume(symbol=context.symbol, volume=context.total, side=PositionSide_Long,
                     order_type=OrderType_Market, position_effect=PositionEffect_Open)
        print(context.symbol, '以市价单开多仓10000股')
        context.first = 1.
        day = bar.bob.strftime('%Y-%m-%d')
        context.day[-1] = day[-2:]
        # 每天的仓位操作
        context.turnaround = [0, 0]
        return
    # 更新最新的日期
    day = bar.bob.strftime('%Y-%m-%d %H:%M:%S')
    context.day[0] = bar.bob.day
    # 若为新的一天,获取可用于回转的昨仓
    if context.day[0] != context.day[-1]:
        context.ending = 0
        context.turnaround = [0, 0]
    if context.ending == 1:
        return
    # 若有可用的昨仓则操作
    if context.total >= 0:
        # 获取时间序列数据
        symbol = bar['symbol']
        recent_data = context.data(symbol=symbol, frequency='60s', count=35, fields='close')
        # 计算MACD线
        macd = talib.MACD(recent_data['close'].values)[0][-1]
        # 根据MACD>0则开仓,小于0则平仓
        if macd > 0:
            # 多空单向操作都不能超过昨仓位,否则最后无法调回原仓位
            if context.turnaround[0] + context.trade_n < context.total:
                # 计算累计仓位
                context.turnaround[0] += context.trade_n
                order_volume(symbol=context.symbol, volume=context.trade_n, side=PositionSide_Long,
                             order_type=OrderType_Market, position_effect=PositionEffect_Open)
                print(symbol, '市价单开多仓', context.trade_n, '股')
        elif macd < 0:
            if context.turnaround[1] + context.trade_n < context.total:
                context.turnaround[1] += context.trade_n
                order_volume(symbol=context.symbol, volume=context.trade_n, side=PositionSide_Short,
                             order_type=OrderType_Market, position_effect=PositionEffect_Close)
                print(symbol, '市价单平多仓', context.trade_n, '股')
        # 临近收盘时若仓位数不等于昨仓则回转所有仓位
        if day[11:16] == '14:55' or day[11:16] == '14:57':
            position = context.account().position(symbol=context.symbol, side=PositionSide_Long)
            if position['volume'] != context.total:
                order_target_volume(symbol=context.symbol, volume=context.total, order_type=OrderType_Market,
                                    position_side=PositionSide_Long)
                print('市价单回转仓位操作...')
                context.ending = 1
        # 更新过去的日期数据
        context.day[-1] = context.day[0]
if __name__ == '__main__':
    '''
    strategy_id策略ID,由系统生成
    filename文件名,请与本文件名保持一致
    mode实时模式:MODE_LIVE回测模式:MODE_BACKTEST
    token绑定计算机的ID,可在系统设置-密钥管理中生成
    backtest_start_time回测开始时间
    backtest_end_time回测结束时间
    backtest_adjust股票复权方式不复权:ADJUST_NONE前复权:ADJUST_PREV后复权:ADJUST_POST
    backtest_initial_cash回测初始资金
    backtest_commission_ratio回测佣金比例
    backtest_slippage_ratio回测滑点比例
    '''
    run(strategy_id='strategy_id',
        filename='main.py',
        mode=MODE_BACKTEST,
        token='token_id',
        backtest_start_time='2017-09-01 08:00:00',
        backtest_end_time='2017-10-01 16:00:00',
        backtest_adjust=ADJUST_PREV,
        backtest_initial_cash=2000000,
        backtest_commission_ratio=0.0001,
        backtest_slippage_ratio=0.0001)
```

## 绩效
![绩效](attach.png)
