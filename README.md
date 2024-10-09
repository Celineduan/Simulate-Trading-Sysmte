# Simulate-Trading-Sysmte
from CAL.PyCAL import *
from datetime import *  
import time
import pandas as pd
import numpy  as np
import math

start = datetime(2014, 1, 1)
end   = datetime(2014, 6, 30)
benchmark = 'HS300'
universe = set_universe('HS300')
capital_base = 100000
refresh_rate = 1
##########################################
t1 = 50    #MA周期
t2 = 30    #ROC周期
MaxBar = 0.75 #持仓周期最大时，首次卖出系数
##########################################
T =  pd.Series(data=[t1],index = universe)
commission = Commission(buycost=0.0003, sellcost=0.0003) 
def initialize(account):
    pass

def handle_data(account):    
    #print account.current_date
    cal = Calendar('China.SSE')
    last_dayt1 = cal.advanceDate(account.current_date, str(-t1)+'B', BizDayConvention.Preceding).toDateTime()           #计算出前t1个交易日
    buylist = []
    selllist = []

    #取出当前交易日至前t1交易日之间的收盘价格数据
    cp = DataAPI.MktEqudGet(secID=account.universe,ticker=u"",tradeDate=u"",beginDate=last_dayt1,endDate=account.current_date,field=[u"tradeDate",u"secID",u"closePrice"],pandas="1")

    #根据持仓周期，更新MA计算周期T，持仓周期增加1，MA计算周期就减少1，最小为10
    for stock in account.avail_secpos:
        T[stock] =  T[stock] - 1
        if(T[stock] < 10):
            T[stock] = 10
    #计算t1周期MA
    cpg = cp['closePrice'].groupby(cp['secID'])
    ma = cpg.mean()
    std = cpg.std()

    #当股票当前价格突破布林线上轨，且ROC值大于0，买入
    for stock in account.universe:
        upband = ma[stock] + std[stock]
        if(len(cp[cp.secID == stock]) > t2):
            roc = (account.referencePrice[stock] - float(cp[cp.secID == stock][-t2:-t2+1]['closePrice'])) / float(cp[cp.secID == stock][-t2:-t2+1]['closePrice'])
            if account.referencePrice[stock] > upband and roc > 0:
                buylist.append(stock)
    #当股票当前价格突破布林线中轨，且ROC值小于0，卖出
    for stock in account.avail_secpos:
        #根据股票的持仓周期计算，MA周期
        MAT = np.mean(cp[cp.secID == stock][-T[stock]:]['closePrice'])
        stdT = np.std(cp[cp.secID == stock][-T[stock]:]['closePrice'])
        midband = MAT
        downband = MAT - stdT
        if(len(cp[cp.secID == stock]) > t2):
            roc = (account.referencePrice[stock] - float(cp[cp.secID == stock][-t2:-t2+1]['closePrice'])) / float(cp[cp.secID == stock][-t2:-t2+1]['closePrice'])
            if account.referencePrice[stock] < midband  and roc < 0:
                selllist.append(stock)
    #买入策略，虚拟账户剩余金额按可买股票平均买入，0.95为成功成交系数
    for i in buylist:
        order(i, account.cash / len(buylist) / account.referencePrice[i] * 0.95)
    #卖出策略，按持仓周期逐步卖出，持仓周期越长，第一次卖出越多，最多为3/4仓位，以10天为单位递减
    for i in selllist:
        x = (account.avail_secpos[i] * MaxBar*math.pow(0.5,T[i] // 10 -1) // 100) * 100
        order(i,-x) 
        if(x == 100):
             T[i] = t1
