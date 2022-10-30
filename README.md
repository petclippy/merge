# Ethereum validators, performing under pressure #


## Obtaining, importing and pre-processing the data ##
Information linking validators to different ethereum validator client software was critical for this analysis. It was made possible by the [blockprint](https://github.com/sigp/blockprint) tool. I obtained the information about the latest predicted beacon node software client used by each validator by querying the `/validator/blocks/latest` endpoint of the blockprint private API. All the following analysis was done in R. The below code takes the json output from the API and creates the lookup-table I used to link validator clients with validator index identifier.
```R
library(jsonlite)
library(data.table)
valCL = fromJSON("C:/R/merge/valCL")
valCL = data.table(val=valCL$proposer_index,cl=valCL$best_guess_single,key="cl")
```
Another important data source for this analysis was information about successful attestations for the finalized chain. This data was extracted from [chaind](https://github.com/wealdtech/chaind). Attestation and proposal success was obtained from the `t_validator_epoch_summaries` table. This table contains one row per validator per epoch. With 450k+ validators, if many epochs are to be analyzed, this table becomes too big for R to handle in memory. My solution was to have a pre-processing step where epochs were imported one by one in memory and the different validator clients aggregated into a new table. This pre-processing code is below. 
```R
library(data.table)
valEpDir = "C:/R/merge/valEp/"
files = list.files(valEpDir)
epDat = list()
for (fileIn in files) {
  ep = fread(paste0(valEpDir,fileIn))
  colnames(ep) = c("f_validator_index","f_epoch","f_proposer_duties","f_proposals_included",
                   "f_attestation_included","f_attestation_target_correct","f_attestation_head_correct",
                   "f_attestation_inclusion_delay","f_attestation_source_timely","f_attestation_target_timely",
                   "f_attestation_head_timely")
  setkey(ep,f_validator_index)
  if (length(unique(ep[["f_epoch"]]))>1) stop(paste("more than 1 epoch in",fileIn))
  for (cl in names(valCList)) {
    valSel = intersect(valCList[[cl]],ep[["f_validator_index"]])
    epCL = ep[.(valSel)]
    attTargHead = epCL[["f_attestation_included"]]=="t" & 
      epCL[["f_attestation_target_correct"]]=="t" & epCL[["f_attestation_head_correct"]]=="t"
    propCount = epCL[["f_proposer_duties"]]==1
    propMiss = epCL[["f_proposer_duties"]]==1 & epCL[["f_proposals_included"]]==0
    inclSum = sum(epCL[["f_attestation_inclusion_delay"]],na.rm=T)-sum(epCL[["f_attestation_included"]]=="t",na.rm=T)
    epDat = c(epDat,list(data.table(Ep=epCL[1][["f_epoch"]],Cl=cl,ClCount=nrow(epCL),
                             InclSum=inclSum / nrow(epCL),
                             PropCount=sum(propCount,na.rm=T),
                             PropRatio=sum(propCount,na.rm=T) / nrow(epCL),
                             PropMiss=sum(propMiss,na.rm=T),
                             PropMissRatio=sum(propMiss,na.rm=T) / sum(propCount,na.rm=T),
                             NotAttTargHead=sum(attTargHead==FALSE,na.rm=T) / nrow(epCL))))
  }
}
epDat = rbindlist(epDat)
epDat[["Slot1"]] = epDat[["Ep"]]*32
```
Another piece of data used in this analysis was attestations at the head-1 slot, obtained by querying my local Lighthouse node every 12 seconds for several weeks. 
```R
forever = TRUE
while (forever==TRUE) {
	st = Sys.time()
	print(paste("Fetching block",st))
	system("curl -X GET 'localhost:5052/eth/v2/debug/beacon/states/head' | jq '{ slot: .data.slot, current_epoch_participation: .data.current_epoch_participation }' > stateAtt")
	att = fromJSON("/home/p/stateAtt")
	stateSlot = as.numeric(att$slot)
	slotSel =  stateSlot-1
	att = data.table(val=seq(0,length(att$current_epoch_participation)-1),attMiss=att$current_epoch_participation=="0",key="val")
	valAtt = merge(valCL,att)
	coms = fromJSON("/home/p/coms")
	if (!(as.character(slotSel) %in% unique(coms$data$slot))) {
		system("curl -X GET 'localhost:5052/eth/v1/beacon/states/head/committees' | jq > coms")
		coms = fromJSON("/home/p/coms")
		firstOfEp = data.table(firstSlotOfEpoch=min(as.numeric(unique(coms$data$slot))))
		fwrite(firstOfEp,file="/home/p/firstOfEp.csv",append=T)
	}
	if (as.character(stateSlot) %in% unique(coms$data$slot) & as.character(slotSel) %in% unique(coms$data$slot)) {
		slotVals = as.numeric(unlist(coms$data$validators[coms$data$slot==slotSel]))
		slotVals = intersect(slotVals,valAtt[["val"]])
		valAttSlot = valAtt[.(slotVals)]
		clMiss = sapply(unique(valAtt[["cl"]]),function(x) sum(valAttSlot[cl==x][["attMiss"]]))
		clSlot = sapply(unique(valAtt[["cl"]]),function(x) nrow(valAttSlot[cl==x]))
		df = data.frame(cl=names(clSlot),sum=as.numeric(clSlot),missSlot=as.numeric(clMiss),percMiss=as.numeric(clMiss)/as.numeric(clSlot)*100)
		slotPerc = data.table(Slot=slotSel,cl=df[,"cl"],percMiss=df[,"percMiss"])
		fwrite(slotPerc,file="/home/p/slotPerc.csv",append=T)
	}
	end = Sys.time()
	print(paste("Block finished",end,"sleep",max(c(0,12 - as.numeric(end-st)))))
	Sys.sleep(time = max(c(0,12 - as.numeric(end-st))))
}
```
The final piece of data used in this analysis was from Flashblots. I downloaded the blocks built by Flashbots during September 2022 from their public API, stored as flashbots_2022-09.csv. This information contained the public key of validators, but not their validator index. To link the validator public key to index, I queried the beaconcha.in public API.
```R
fb = fread("C:/R/merge/flashbots_2022-09.csv")
fbChunks = split(fb[["proposer_pubkey"]], ceiling(seq_along(fb[["proposer_pubkey"]])/20))
for (chunkInd in 1:length(fbChunks)) {
  print(paste("Starting",chunkInd))
  slotStr = paste(fbChunks[[chunkInd]],collapse="%2C")
  tmp = data.table(fromJSON(paste0("https://beaconcha.in/api/v1/validator/",slotStr))$data)
  fwrite(tmp,file="C:/R/merge/flashbots_2022-09_beaconChain.csv",append=T)
  Sys.sleep(10)
}
```


## Plotting data ##
The first plot is of the distribution of different validator clients, using the latest measurement from blockprint.
```R
library(ggplot2)
library(treemapify)
df = data.frame(table(valCL[["cl"]]))
df[,"perc"] = signif(df[,"Freq"] / sum(df[,"Freq"]) * 100,3)
df = df[c(1,3,4,5,2),]
p = ggplot(df, aes(area = Freq, fill=factor(Var1,levels=unique(Var1)),label = paste(Var1,paste(Freq,"validators"),paste(perc,"%"),sep="\n"))) +
  geom_treemap() +
  geom_treemap_text(colour = "white",place = "centre",size = 10) +
  scale_fill_brewer(palette = "Set2") +
  theme(legend.position="none")
ggsave(filename=paste0("C:/R/merge/valCL_treemap.png"),p,width=6,height=3)
```
![valCL_treemap](https://github.com/petclippy/merge/blob/main/valCL_treemap.png?raw=true)


The second plot shows the % of validators not successfully participating at the finalized head of the chain:
```R
library(ggplot2)
p = ggplot(epDat,aes(Ep,NotAttTargHead*100,color=Cl)) +
  scale_colour_brewer(palette = "Set2") +
  geom_smooth(method="loess",span=0.15,se=F,size=1.5) +
  theme_bw() +
  geom_vline(xintercept=c(144896,146875), linetype="dotted",color="red",size=1) +
  geom_text(aes(x=144896+120, label="Bellatrix", y=3), colour="red", angle=90, size=6) +
  geom_text(aes(x=146875+120, label="Merge", y=3), colour="red", angle=90, size=6) +
  labs(title="Not attesting productively. Aug. 24 to Sept. 28, 2022",
       x="Epoch",y="% not attesting",color="Validator client")
ggsave(filename=paste0("C:/R/merge/notAttTargHeadCL.png"),p,width=10,height=4)
```
![notAttTargHeadCL](https://github.com/petclippy/merge/blob/main/notAttTargHeadCL.png?raw=true)


The third and fourth plots are animations. They were made by looping through epochs to create one png to per epoch, then combining png's using the [gifski](https://github.com/ImageOptim/gifski) tool. To speed up the animation in the less interesting intervals and slow them down during Bellatrix or the merge, different intervals between making .png's was used:
```R
library(ggplot2)
ep142000date = "2022-08-24"
ep142000time = "14:40:23"
ep14200 = as.POSIXct(paste(ep142000date, ep142000time),format = "%Y-%m-%d %H:%M:%S",tz="UTC")
eps = seq(143000,144850,by=6)
for (ep in eps) {
  pTmp = pT[pT[,"Ep"] > ep-100 & pT[,"Ep"] <= ep,]
  pTmp[,"alph"] = (pTmp[,"Ep"]-min(pTmp[,"Ep"])) / (max(pTmp[,"Ep"])-min(pTmp[,"Ep"]))
  pTmp[,"NotAttTargHead"] = pTmp[,"NotAttTargHead"] * 100
  p = ggplot(pTmp,aes(Cl,NotAttTargHead,color=Cl)) +
    scale_colour_brewer(palette = "Set2") +
    theme_bw() +
    geom_quasirandom(size=1,aes(alpha=alph)) +
    ylim(0,16) +
    theme(legend.position="none",plot.title = element_text(size=12),axis.title.x=element_blank()) +
    labs(title=paste("Epoch:",ep," | ",ep14200+(ep-142000)*12*32,"UTC"),y="% not attesting")
  ggsave(filename=paste0("C:/R/merge/plot100-6_",min(eps),"-",max(eps),"/",ep,".png"),p,width=5,height=4)
}
eps = seq(144851,145200,by=1)
for (ep in eps) {
  pTmp = pT[pT[,"Ep"] > ep-100 & pT[,"Ep"] <= ep,]
  pTmp[,"alph"] = (pTmp[,"Ep"]-min(pTmp[,"Ep"])) / (max(pTmp[,"Ep"])-min(pTmp[,"Ep"]))
  pTmp[,"NotAttTargHead"] = pTmp[,"NotAttTargHead"] * 100
  p = ggplot(pTmp,aes(Cl,NotAttTargHead,color=Cl)) +
    scale_colour_brewer(palette = "Set2") +
    theme_bw() +
    geom_quasirandom(size=1,aes(alpha=alph)) +
    ylim(0,16) +
    theme(legend.position="none",plot.title = element_text(size=12),axis.title.x=element_blank()) +
    labs(title=paste("Epoch:",ep," | ",ep14200+(ep-142000)*12*32,"UTC"),y="% not attesting")
  ggsave(filename=paste0("C:/R/merge/plot100-1_",min(eps),"-",max(eps),"/",ep,".png"),p,width=5,height=4)
}
eps = seq(145201,146850,by=8)
for (ep in eps) {
  pTmp = pT[pT[,"Ep"] > ep-100 & pT[,"Ep"] <= ep,]
  pTmp[,"alph"] = (pTmp[,"Ep"]-min(pTmp[,"Ep"])) / (max(pTmp[,"Ep"])-min(pTmp[,"Ep"]))
  pTmp[,"NotAttTargHead"] = pTmp[,"NotAttTargHead"] * 100
  p = ggplot(pTmp,aes(Cl,NotAttTargHead,color=Cl)) +
    scale_colour_brewer(palette = "Set2") +
    theme_bw() +
    geom_quasirandom(size=1,aes(alpha=alph)) +
    ylim(0,16) +
    theme(legend.position="none",plot.title = element_text(size=12),axis.title.x=element_blank()) +
    labs(title=paste("Epoch:",ep," | ",ep14200+(ep-142000)*12*32,"UTC"),y="% not attesting")
  ggsave(filename=paste0("C:/R/merge/plot100-8_",min(eps),"-",max(eps),"/",ep,".png"),p,width=5,height=4)
}
eps = seq(146851,147250,by=1)
for (ep in eps) {
  pTmp = pT[pT[,"Ep"] > ep-100 & pT[,"Ep"] <= ep,]
  pTmp[,"alph"] = (pTmp[,"Ep"]-min(pTmp[,"Ep"])) / (max(pTmp[,"Ep"])-min(pTmp[,"Ep"]))
  pTmp[,"NotAttTargHead"] = pTmp[,"NotAttTargHead"] * 100
  p = ggplot(pTmp,aes(Cl,NotAttTargHead,color=Cl)) +
    scale_colour_brewer(palette = "Set2") +
    theme_bw() +
    geom_quasirandom(size=1,aes(alpha=alph)) +
    ylim(0,16) +
    theme(legend.position="none",plot.title = element_text(size=12),axis.title.x=element_blank()) +
    labs(title=paste("Epoch:",ep," | ",ep14200+(ep-142000)*12*32,"UTC"),y="% not attesting")
  ggsave(filename=paste0("C:/R/merge/plot100-1_",min(eps),"-",max(eps),"/",ep,".png"),p,width=5,height=4)
}
eps = seq(147251,149842,by=8)
for (ep in eps) {
  pTmp = pT[pT[,"Ep"] > ep-100 & pT[,"Ep"] <= ep,]
  pTmp[,"alph"] = (pTmp[,"Ep"]-min(pTmp[,"Ep"])) / (max(pTmp[,"Ep"])-min(pTmp[,"Ep"]))
  pTmp[,"NotAttTargHead"] = pTmp[,"NotAttTargHead"] * 100
  p = ggplot(pTmp,aes(Cl,NotAttTargHead,color=Cl)) +
    scale_colour_brewer(palette = "Set2") +
    theme_bw() +
    geom_quasirandom(size=1,aes(alpha=alph)) +
    ylim(0,16) +
    theme(legend.position="none",plot.title = element_text(size=12),axis.title.x=element_blank()) +
    labs(title=paste("Epoch:",ep," | ",ep14200+(ep-142000)*12*32,"UTC"),y="% not attesting")
  ggsave(filename=paste0("C:/R/merge/plot100-8_",min(eps),"-",max(eps),"/",ep,".png"),p,width=5,height=4)
}
```
![plotBella](https://github.com/petclippy/merge/blob/main/plotBella.gif?raw=true)
![plotMerge8](https://github.com/petclippy/merge/blob/main/plotMerge8.gif?raw=true)


The fifth plot shows the head-1 attestation participation for slots as they are seen at the head:
```R
library(data.table)
library(ggplot2)
slotPerc = fread("C:/R/merge/slotPerc.csv",key="Slot")
slotPerc = unique(slotPerc)
slotMissSum = sapply(slotPerc[["Slot"]],function(x) sum(slotPerc[.(x)][["percMiss"]]))
slotSel = slotMissSum > 1 & slotMissSum < 100
slotPerc = slotPerc[slotSel]
firstOfEp = fread("C:/R/merge/firstOfEp.csv")[[1]]
firstOfEp = intersect(firstOfEp,slotPerc[["Slot"]])
postMergSlot = 4700032
pT = list()
for (fSlot in firstOfEp) {
  slots = c(fSlot,fSlot+seq(1,31))
  slots = intersect(slots,slotPerc[["Slot"]])
  if (length(slots)<5) next
  slotInd = slots-fSlot+1
  perc = unique(slotPerc[.(slots)])
  for (client in c("Lighthouse","Nimbus","Prysm","Teku")) {
    pT = c(pT,list(data.table(Slot=slots,SlotInd=slotInd,Cl=client,PercMiss=perc[cl==client][["percMiss"]])))
  }
}
pT = rbindlist(pT)
p = ggplot(pT[order(SlotInd)][Slot>postMergSlot][SlotInd<6],aes(factor(SlotInd,levels=unique(SlotInd)),PercMiss,fill=Cl)) +
  scale_fill_brewer(palette = "Set2") +
  geom_boxplot(outlier.shape=NA) +
  theme_bw() +
  ylim(0,10) +
  labs(title="After merge",x="Slot position inside epoch",y="% not attested for head-1 slot",fill="Validator client")
ggsave(filename=paste0("C:/R/merge/headAtt_boxes_postMerg_below6.pdf"),p,width=5,height=3)
p = ggplot(pT[order(SlotInd)][Slot<postMergSlot][SlotInd<6],aes(factor(SlotInd,levels=unique(SlotInd)),PercMiss,fill=Cl)) +
  scale_fill_brewer(palette = "Set2") +
  geom_boxplot(outlier.shape=NA) +
  theme_bw() +
  ylim(0,10) +
  labs(title="Before merge",x="Slot position inside epoch",y="% not attested for head-1 slot",fill="Validator client")
ggsave(filename=paste0("C:/R/merge/headAtt_boxes_beforeMerg_below6.pdf"),p,width=5,height=3)
```
![headAtt_boxes_beforeMerg_below6](https://github.com/petclippy/merge/blob/main/headAtt_boxes_beforeMerg_below6.png?raw=true)
![headAtt_boxes_postMerg_below6](https://github.com/petclippy/merge/blob/main/headAtt_boxes_postMerg_below6.png?raw=true)


The sixth plot shows % missed block proposals:
```R
library(ggplot2)
p = ggplot(epDat,aes(Ep,PropMissRatio*100,group=Cl,color=Cl)) +
  geom_smooth(method="loess",span=0.25,se=F,size=2,alpha=0.2) +
  theme_bw() +
  scale_colour_brewer(palette = "Set2") +
  labs(title="Missed block proposals. Aug. 24 to Sept. 28, 2022",
       y="% missed proposals",
       x="Epoch",color="Validator client") +
  geom_vline(xintercept=c(144896,146875), linetype="dotted",color="red",size=1) +
  geom_text(aes(x=144896+120, label="Bellatrix", y=3), colour="red", angle=90, size=6) +
  geom_text(aes(x=146875+120, label="Merge", y=3), colour="red", angle=90, size=6)
ggsave(p,file="C:/R/merge/propMissEp.png",width=10,height=4)
```
![propMissEp](https://github.com/petclippy/merge/blob/main/propMissEp.png?raw=true)


The seventh plot shows the usage of the Flashblots relay:
```R
library(data.table)
library(ggplot2)
fb = fread("C:/R/merge/flashbots_2022-09.csv")
fbSl = unique(fread("C:/R/merge/flashbots_2022-09_beaconChain.csv",key="pubkey"))
fb[["val"]] = sapply(fb[["proposer_pubkey"]],function(x) as.numeric(fbSl[.(x)][1][["validatorindex"]]))
fb[["cl"]] = sapply(fb[["val"]],function(x) as.character(valCL[.(x)][["cl"]]))
fb[["ep"]] = floor(fb[["slot"]]/32)
pT = list()
for (epSel in unique(epDat[["Ep"]])) {
  fbepTmp = fb[ep==epSel]
  epDatTmp = epDat[Ep==epSel]
  for (clSel in unique(epDatTmp[PropCount>0][["Cl"]])) {
    percFB = nrow(fbepTmp[cl==clSel]) / epDatTmp[Cl==clSel][["PropCount"]] * 100
    pT = c(pT,list(data.table(Ep=epSel,Cl=clSel,PercFb=percFB)))
  }
}
pT = rbindlist(pT)
p = ggplot(pT[Ep>146875],aes(Ep,PercFb,group=Cl,color=Cl)) +
  geom_smooth(method="loess",span=0.2,size=2,se=F,alpha=0.25) +
  theme_bw() +
  labs(title="Validator clients block proposals with MEV through Flashbots. Sept. 15 to Oct. 1, 2022",
       x="Epoch",
       y="% of blocks proposed built by Flashbots",
       color="Validator client") +
  scale_colour_brewer(palette = "Set2") +
  geom_vline(xintercept=146875,linetype="dotted",color="red",size=1) +
  geom_text(aes(x=146875+45,label="Merge", y=(30)), colour="red", angle=90, size=6)
ggsave(p,file="C:/R/merge/fbEpClProp.png",width=10,height=4)
```
![fbEpClProp](https://github.com/petclippy/merge/blob/main/fbEpClProp.png?raw=true)


The eighth and final (bonus) plot shows the luck of validator using different clients for getting block proposals:
```R
library(ggplot2)
p = ggplot(epDat,aes(Ep,PropRatio*100,group=Cl,color=Cl)) +
  geom_smooth(method="loess",span=0.25,se=F,size=2,alpha=0.2) +
  theme_bw() +
  scale_colour_brewer(palette = "Set2") +
  labs(title="Block proposal likelyhood. Sept. 2 to Sept. 28, 2022",
       y="% likelihood of proposing block in an epoch",
       x="Epoch",color="Validator client") +
  geom_vline(xintercept=c(144896,146875), linetype="dotted",color="red",size=1) +
  geom_text(aes(x=144896+70, label="Bellatrix", y=0.0078), colour="red", angle=90, size=6) +
  geom_text(aes(x=146875+70, label="Merge", y=0.0078), colour="red", angle=90, size=6) +
  xlim(144896,NA)
ggsave(p,file="C:/R/merge/propLuck.png",width=10,height=4)
```
![propLuck](https://github.com/petclippy/merge/blob/main/propLuck.png?raw=true)
