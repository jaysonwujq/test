## 多因素

![Error](https://github.com/bigone1/test/blob/master/Screenshots/10.png)

```R
data <- read.table('D:/linshiwenjian/fanai/CRC/multivariate_OS.txt',sep='\t',header = T,stringsAsFactors = F)
f <- Surv(OS_time,OS_status)~age_group+exp_group+Gender+stage_group+N_group+T_group+PM
m <- coxph(f,data = data)
summary(m)
```

## 森林图

![Error](https://github.com/bigone1/test/blob/master/Screenshots/11.png)

```R
library(forestplot)
data <- read.csv("D:/linshiwenjian/fanai/CRC/forestdata.csv", stringsAsFactors=FALSE)

#简单森林图
## 构建tabletext，更改列名称，展示更多信息
np <- ifelse(!is.na(data$Count), 
             paste(data$Count,sep=""), NA)

## The rest of the columns in the table.
tabletext <- cbind(c("","\n",data$Variable),
                   c("P Value","\n",data$P.Value))
##绘制森林图
forestplot(labeltext=tabletext, graph.pos=3,
           mean=c(NA,NA,data$Point.Estimate),
           lower=c(NA,NA,data$Low), upper=c(NA,NA,data$High),
           boxsize=0.5)

#优化森林图
## 定义亚组，方便后面线条区分
subgps <- c(2,3,6,7,10,11,14,15,18,19,22,23,26,27)
data$Variable[subgps] <- paste("  ",data$Variable[subgps])

forestplot(labeltext=tabletext,
           graph.pos=3, #为Pvalue箱线图所在的位置
           mean=c(NA,NA,data$Point.Estimate),
           lower=c(NA,NA,data$Low), upper=c(NA,NA,data$High),
           #定义标题
           title="COAD",
           xticks = c(0,0.5,1,16),
           ##定义x轴
           xlab="    <---Better OS---   ---Worse OS--->",
           ##根据亚组的位置，设置线型，宽度造成“区块感”
           #hrzl_lines=list("2" = gpar(lwd=1, col="#99999922"),
                           #"6" = gpar(lwd=60, lineend="butt", columns=c(2:6), col="#99999922"),
                           #"14" = gpar(lwd=60, lineend="butt", columns=c(2:6), col="#99999922"),
                           #"22" = gpar(lwd=60, lineend="butt", columns=c(2:6), col="#99999922")),
           hrzl_lines=list("3" = gpar(lwd=2, col="black"),#第三行顶部加黑线，引号内数字标记行位置
                           #"4" = gpar(lwd=60,lineend="butt", columns=c(1:4), col="#99999922"),#加阴影，弱项不建议使用
                           "30" = gpar(lwd=2, col="black")),#最后一行底部加黑线,""
           #fpTxtGp函数中的cex参数设置各个组件的大小
           txt_gp=fpTxtGp(label=gpar(cex=1.25),
                     ticks=gpar(cex=1.1),
                     xlab=gpar(cex = 1.2),
                     title=gpar(cex = 1.2)),
           #mar=unit(rep(1.25, times = 4), "cm"),
           ##fpColors函数设置颜色
           fn.ci_norm="fpDrawDiamondCI",
           col=fpColors(box="steelblue", lines="black", zero = "black"),
           #箱线图中基准线的位置        
           zero=1,
           cex=0.9, 
           lwd.xaxis=2,
           lineheight = unit(7,'mm'),
           colgap=unit(4,"mm"),
           #箱子大小，线的宽度
           lwd.ci=2, 
           boxsize=0.5,
           #箱线图两端添加小竖线，高度
           ci.vertices=TRUE, 
           ci.vertices.height = 0.4
           )
```

## Cox回归分析——森林图

```R
#step1   生成示例数据
options(stringsAsFactors = F)
# 生成模拟数据(基因表达与生存时间)
df <- data.frame(ID = paste0("TCGA-",sample(100,100,replace = F)),
                 time = abs(rnorm(100, mean = 2, sd = 5)),
                 status = sample(c(0,1),100,replace = T))
geneVec <- replicate(50,paste0(sample(LETTERS,4,replace = T),collapse = ""))

for (i in geneVec) {
  dfGene <- data.frame(i = abs(rnorm(100, mean = 5, sd = 8)))
  df <- cbind(df,dfGene)
}
colnames(df)[4:length(df)] <- geneVec

#step2 多因素Cox回归
# COX模型构建
library(survival)
multiCox <- coxph(Surv(time, status) ~ ., data = df[,-1])
multiCox <- step(multiCox,direction = "both")
multiCoxSum <- summary(multiCox)
# 输出模型参数
out_multi <- data.frame()
out_multi <- cbind(
  coef=multiCoxSum$coefficients[,"coef"],
  HR=multiCoxSum$conf.int[,"exp(coef)"],
  HR.95L=multiCoxSum$conf.int[,"lower .95"],
  HR.95H=multiCoxSum$conf.int[,"upper .95"],
  pvalue=multiCoxSum$coefficients[,"Pr(>|z|)"])
out_multi <- as.data.frame(cbind(id=row.names(out_multi),out_multi))

#step3 森林图
# 森林图
out_multi[,2:ncol(out_multi)] <- as.numeric(unlist(out_multi[,2:ncol(out_multi)]))
hz <- paste(round(out_multi$HR,3),
            "(",round(out_multi$HR.95L,3),
            "-",round(out_multi$HR.95H,3),")",sep = "")
tabletext <- cbind(c(NA,"Gene",out_multi$id),
                   c(NA,"Coefficient",round(out_multi$coef,3)),
                   c(NA,"P value",ifelse(out_multi$pvalue<0.001,"P < 0.001",round(out_multi$pvalue,3))),
                   c(NA,"Hazard Ratio(95% CI)",hz))
nrow(tabletext)+1
library(forestplot)
forestplot(labeltext=tabletext, 
           graph.pos=3,  #为Pvalue箱线图所在的位置
           col=fpColors(box="#D55E00", lines="#CC79A7", zero = "gray50"),
           mean=c(NA,NA,out_multi$HR),
           lower=c(NA,NA,out_multi$HR.95L), #95%置信区间下限
           upper=c(NA,NA,out_multi$HR.95H), #95%置信区间上限
           boxsize=0.3,lwd.ci=2,   #箱子大小，线的宽度
           ci.vertices.height = 0.08,ci.vertices=TRUE, #置信区间用线宽、高、型
           zero=1,lwd.zero=1,      #zero线宽 基准线的位置
           colgap=unit(5,"mm"),    #列间隙
           xticks = c(0.5, 1,1.5), #横坐标刻度
           lwd.xaxis=1,            #X轴线宽
           lineheight = unit(0.8,"cm"), #固定行高
           graphwidth = unit(.3,"npc"), #图在表中的宽度比例
           cex=0.9, fn.ci_norm = fpDrawCircleCI, #误差条显示方式
           hrzl_lines=list("2" = gpar(lwd=2, col="black"),
                           "3" = gpar(lwd=2, col="black"), #第三行顶部加黑线，引号内数字标记行位置
                           "34" = gpar(lwd=2, col="black")),#最后一行底部加黑线,"34"中数字为nrow(tabletext)+1
           mar=unit(rep(0.5, times = 4), "cm"),#图形页边距
           #fpTxtGp函数中的cex参数设置各个组件的大小
           txt_gp=fpTxtGp(label=gpar(cex=1),
                          ticks=gpar(cex=1.5),
                          xlab=gpar(cex = 1.25),
                          title=gpar(cex = 1.2)),
           xlab="Hazard Ratio")
```
