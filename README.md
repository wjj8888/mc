# coding=utf-8
from time import time

import numpy as np
import pandas as pd
from pulp import *


def read_data(file):#从数据中提取供给或需求点的坐标
    df = pd.read_csv(file, encoding='utf-8')
    df = df.reset_index(drop=True)
    return df
def deal_od(od_data,data_demand,data_supply):
    num_demand=len(data_demand)
    num_supply=len(data_supply)
    dis=np.array(od_data['distance'])
    dem_index=data_demand.index.tolist()
    sup_index = data_supply.index.tolist()
    dis=dis.reshape(num_demand,num_supply)
    df_od=pd.DataFrame(dis,index=dem_index,columns=sup_index)
    return df_od
def MCLPModel(I,J,a,N,H):
    variables={}
    prob = LpProblem("MCLP", LpMaximize)
    #设置决策变量
    # for j in J:
    #     #if j<208:
    #     if j < 149:
    #         variables[j] = LpVariable("x_"+str(j), lowBound = 0, upBound = 1, cat="Integer")
    #     else:
    #         variables[j]=1
    for j in J:
        # if j<208:
        variables[j] = LpVariable("x_" + str(j), lowBound=0, upBound=1, cat="Integer")
    y = LpVariable.dicts("y", I, lowBound = 0, upBound = 1, cat="Integer")
    # Maximizes
    #设置目标函数
    prob += lpSum([a[i]*y[i] for i in I])
    #设置约束条件
    #prob += lpSum([variables[j] for j in range(194, 213, 1)]) == 19 # 保留推荐
    #prob += lpSum([variables[j] for j in range(138, 213, 1)]) == 75  # 保留全部75
    for i in I:
        prob += lpSum([variables[j] for j in N[i]]) >= y[i]
    prob += lpSum([variables[j] for j in J]) == H
    time_start = time()
    #模型求解
    prob.solve()
    time_end = time()
    tot_cost=str(time_end-time_start)
    print(str(LpStatus[prob.status]))
    print(str(value(prob.objective)))
    result_list=[]
    #求解结果分析
    for j in J:
        if isinstance(variables[j],int):
            if variables[j]==1:
                print(str(j)+":")
                print(variables[j])
                result_list.append(j)
        else:
            if variables[j].varValue==1:
                print(str(j)+":")
                print(variables[j].varValue)
                result_list.append(j)
    print(result_list)
    print("求解时间为："+ tot_cost+'s')
    # for v in prob.variables():
    #     print(v)
    return  result_list
def get_selectedOD(distance,results_list,path_od):
    selectedOD=distance[results_list]
    ODarray=selectedOD.values.flatten()
    df=pd.DataFrame()
    df['distance']=ODarray.tolist()
    df.to_csv(path_od,index=False,encoding='utf-8')
    print(selectedOD)
    print(df)
def writeResult(result, prob, facilities, a):
    result.write("Population Served is = %s\n" %value(prob.objective))
    result.write("The Ratio of covered population is = %s" %round(value(prob.objective)/sum(a) * 100, 1))
    result.write("% \n")
    result.write("location = %s\n\n" %facilities[0])
def readresults(sites,results_list,result_path):
    results=[]
    for rt in results_list:
        result=sites.iloc[rt].tolist()
        result.insert(0,rt)
        results.append(result)
    col=['Name','Num','Latitude', 'Longitude']
    dflist = pd.DataFrame(results,columns=col)
    # dflist=dflist.drop(['FID'],axis=1)
    dflist.to_csv(result_path,index=False,encoding='utf-8')
    print(dflist)
# variable setting
def setpath(filepath):
    path=os.path.dirname(filepath)
    print(path)
    file_name = os.path.basename(filepath).split('.')[0]
    output_name_OD=file_name+"_"+str(H)+"OD.csv"
    output_name_sites=file_name+"_"+str(H)+".csv"
    outputpath_OD=os.path.join(path,output_name_OD)
    outputpath_sites=os.path.join(path,output_name_sites)
    return outputpath_OD,outputpath_sites
if __name__ == '__main__':
    demands_file = r"G:\data\**小区.csv"
    sites_file = r"G:\data\**候选点.csv"
    od_file = r"G:\data\**候选点OD.csv"
    demands = read_data(demands_file)  # demand nodes
    sites = read_data(sites_file)  # facility sites
    od = read_data(od_file)
    I = demands.index.tolist()
    J = sites.index.tolist()
    S = 1080
    a = demands['Num']
    distance = deal_od(od, demands, sites)
    N = [[j for j in J if distance[j][i] <= S] for i in I]
    for H in range(125, 145, 10):
        path_od, result_path = setpath(sites_file)
        result_lists = MCLPModel(I, J, a, N, H)
        readresults(sites, result_lists, result_path)
        get_selectedOD(distance, result_lists, path_od)
        print("done")
#writeResult(result, prob, facilities, a)
#result.close()
