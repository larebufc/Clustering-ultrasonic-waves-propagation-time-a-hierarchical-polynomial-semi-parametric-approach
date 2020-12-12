


### Clustering ultrasonic waves propagation time: a hierarchical polynomial semi-parametric approach



In this material, the scripts for implement the methodology presented in the manuscript: Clustering ultrasonic waves propagation time: a hierarchical polynomial semi-parametric approach.  This work is the result of a partnership between the Federal University of Ceará and the Federal University of São Calos,  Brazil. The experiment was carried out at the Laboratory of the Reabilitation and Durability of Constructions (LAREB),  <https://lareb.ufc.br/>, in partnership with the Laboratory of Innovative Technologies (from portuguese LTI) <https://lti.ufc.br/>, both belonging to the Federal University of Ceará.

Autors: Daiane Aparecida Zuanetti, Rosineide da Paz,  Talisson Rodrigues  and Esequiel Mesquita.




The results and the codes for all realized analyses are shown in RStudio Connect in the follow links.

[Sensitivity analysis: normal distribution prior for parameter](https://beta.rstudioconnect.com/content/14658)


The code for the final model for the simultated data is shown below.

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```


## __Analysis of contamined simulated data set.__

In this work, a data-driven hierarchical regression model is applied to simulated data set. 



#### R packages required.

```{r echo=TRUE, paged.print=TRUE}
if(!require(mvtnorm)) install.packages("mvtnorm");require(mvtnorm) 
if(!require(MCMCpack)) install.packages("MCMCpack");require(MCMCpack) 
```


#### Definition of the simulated global population

```{r}
m=300    #number of individuals
Tp=5     #number of replications for each individual
n=m*Tp   #total number of observations
K=4      #number of groups with different effects
sem=100  ## seed
pcov<-2  # Total of fixed effects
q<-pran<-3  # Total of random effects
indiv=sort(rep(1:m,Tp)) # Information about individuals
repl=table(indiv)       # Information about replication in the individuals
#

```



#### Function for simulating indicator variables from a discrete distribution

```{r}
rDiscreta<-function(p){
 u<-runif(1)
 P<-cumsum(p)
 val<-sum(P<u)+1
 val}
#
pgrupos<-c(0.20,0.40,0.20,0.20)   ## Weight or probability for each group
set.seed(sem)
Sverd<-numeric(m)
for (i in 1:m){
	Sverd[i]<-rDiscreta(pgrupos)   ## Indicator group variable
}


```



#### Parameters for the fixed-effect

```{r}
set.seed(sem)
bfixo_verd =matrix(rnorm(pcov,0,1),ncol=1)
beta=c("beta1","beta2")
row.names(bfixo_verd)=beta


```



#### Parameters for the random-effect.


```{r}
a1=c(-4,-2,1,6)
a2=c(2,1.5,5,2)
set.seed(sem) 
sigm=diag(a2[1],q,q)
b1=rmvnorm(1,rep(a1[1],pran),sigm)
sigm=diag(a2[2],q,q)
b2=rmvnorm(1,rep(a1[2],pran),sigm)
sigm=diag(a2[3],q,q)
b3=rmvnorm(1,rep(a1[3],pran),sigm)
sigm=diag(a2[4],q,q)
b4=rmvnorm(1,rep(a1[4],pran),sigm)
#
Buv=t(rbind(b1,b2,b3,b4)) ## unique random-effect matrix
B<-NULL
for (i in 1:m){
	b<-Buv[,Sverd[i]]
	B=rbind(B,b)
}
```






#### Explained  variables for fixed-effect for each individual




```{r}
set.seed(sem)
X<-matrix(0,nrow=m,ncol=pcov)
X[,1]<-runif(m,0,1)
X[,2]<-rnorm(m)
X1f<-NULL
X2f<-NULL
for (i in 1:m){
  X1f<-c(X1f,rep(X[i,1],repl[i]))
  X2f<-c(X2f,rep(X[i,2],repl[i]))}
X<-cbind(X1f,X2f)
```



#### Simulated non-observed random variable for each group

## Random variable represent the effect of the group para Simulated non-observed random variables for each group, which represent the effects of the group.

```{r}
Vt<-1:Tp
aux=matrix(round(poly(Vt, degree=3),3),ncol=3)
aux<-cbind(1,aux)
Z<-NULL
for (i in 1:m) Z<-rbind(Z,aux)
Z=Z[,-1]
head(X); head(Z)
```





#### Generated response variable


```{r}
Y=NULL
set.seed(sem)
err=matrix(rnorm(n,0,1),m,Tp)
for(i in 1:m){
 	Y<-rbind(Y,(as.matrix(X[which(indiv==i),],nrow=sum(indiv==i))%*%bfixo_verd+
 	              Z[which(indiv==i),]%*%matrix(Buv[,Sverd[i]],ncol=1)+err[i,]) )}
#
nout=0  # if nout > 0 outliers are included in the sample
if(nout==0){sout=NULL}else{sout=1:nout
out=numeric()
set.seed(sem)
for(r in 1:nout){	out=rbind(out,(as.matrix(X[which(indiv==i),],nrow=sum(indiv==i))%*%bfixo_verd+
	              log(c(1,2,3,4,5))*(c(1,3,6,3,-8))  + rnorm(5,0,5)) )}
 Y[1:25]=out
}
#
ini=nout+1
plot(NULL, xlim=c(1,Tp), ylim=c(-15,15), ylab="Response", xlab="Index")
for (i in sout) lines(1:Tp,Y[indiv==i],col=(max(Sverd)+2),lty=(max(Sverd)+2))
for (i in ini:m) lines(1:Tp,Y[indiv==i],col=Sverd[i],lty=Sverd[i])
legend(1, 13.5, box.lty=0,legend=c("K = 1", "K = 2", "K = 3", "K = 4", "Outlier"),
       col=c(unique(Sverd),(max(Sverd)+2)),lty=c(unique(Sverd),(max(Sverd)+2)), cex=0.8,horiz=TRUE)
#
```




#### Function for sampling from the posterior of parameter $\sigma^2$

```{r}
poster_sigma2<-function(neta.a,neta.b,residuos){
 alpha<-(length(residuos)/2)+neta.a
 beta<-(sum(residuos^2)/2)+neta.b
 sigma2<-1/(rgamma(1,alpha,beta))
 return(sigma2)}
```


#### Function for sampling fro posterior distribution of parameter $\alpha$.

```{r}
gera_eta_alpha<-function(alpha,a_alph_prior,b_alph_prior,K,m){
	eta_aux<-rbeta(1,alpha+1,m)
	aux_prob<-(a_alph_prior+K-1)/(m*(b_alph_prior-log(eta_aux)))
	prob_alpha<-aux_prob/(1+aux_prob)
	unif<-runif(1)
	if (unif<=prob_alpha) alpha<-rgamma(1,a_alph_prior+K,b_alph_prior-log(eta_aux)) else alpha<-rgamma(1,a_alph_prior+K-1,b_alph_prior-log(eta_aux))
	return(alpha)}
```



##### Hyperparameters of the model

```{r}
lambda1<-lambda2<-0.01
qcov<-ncol(Z)
ni<-4 # degree of freedom of inverse-wishart
sigma<-diag(100,ncol=ncol(Z),nrow=ncol(Z)) #  variable and explained variable of inverse-wishart
for (i in 1:ncol(Z)){
	for (j in 1:ncol(Z)) if (sigma[i,j] != 100) sigma[i,j]<-0.5}

a_alph_prior<-b_alph_prior<-1
```


#### Useful informations for MCMC iterations.

```{r}
XXinv<-solve(t(X)%*%X)
Xtrans<-t(X)
XXinvXtrans<-XXinv%*%Xtrans
```


#### Initial values for  the MCMC chain



```{r}
library(mvtnorm)
library(MCMCpack)
#
set.seed(100)
S_vig<-rep(1,m)
S_obs_vig<-NULL
for (i in 1:m) S_obs_vig<-c(S_obs_vig,rep(S_vig[i],repl[i]))
K<-length(table(S_vig))
sigma2_vig<-round(poster_sigma2(lambda1,lambda2,c(Y)),3)
beta_vig<-matrix(0,ncol=1,nrow=qcov)
Dmat<-riwish(ni,sigma)
b_vig<-matrix(0,ncol=K,nrow=qcov)
for (k in 1:K){
	Bk<-Z[which(S_obs_vig==k),]
	aux<-solve((t(Bk)%*%Bk)+(solve(Dmat)*sigma2_vig))
	media<-aux%*%((t(Bk)%*%Y[which(S_obs_vig==k),])+(sigma2_vig*solve(Dmat)%*%beta_vig))
	varcov<-aux*sigma2_vig
	b_vig[,k]<-t(rmvnorm(1, mean = c(media), sigma = varcov, method=c("chol"), pre0.9_9994 = TRUE))}
ran_pred<-NULL
for (i in 1:n) ran_pred[i]<-Z[i,]%*%matrix(b_vig[,S_obs_vig[i]],ncol=1)
residuos<-matrix(Y-ran_pred,ncol=1)
gama_vig<-rmvnorm(1,mean = c(XXinvXtrans%*%residuos), sigma=(XXinv*sigma2_vig), method=c("chol"), pre0.9_9994 = TRUE)
alpha_vig<-gera_eta_alpha(1,a_alph_prior,b_alph_prior,K,m)
#
```



#### Definition for MCMC convergency.

```{r}
burnin<-500         ## burnig
amostrasfin<-1500    ## sample used for inference
saltos<-3            ## jump definition    
```


#### MCMC function for MCMC runing



```{r echo=TRUE, message=FALSE, warning=FALSE}
est.param <- list(sigma2=NULL, 
                  bs=list(),
                  Sj=NULL, 
                  Gamma=NULL, 
                  K=NULL, 
                  alpha=NULL, 
                  Betas=NULL, 
                  matrix_D=NULL,
                  log_vero=NULL)		
AmostrasTotal<-burnin+amostrasfin*saltos
K=1

set.seed(sem)

for (int in 1:AmostrasTotal){
#for (int in 1:8){
	cat('\n',c(int,K))
	#
	######## update S
	#
	for (i in 1:m){
		nk<-numeric(K)
		prob<-numeric(K)
		for (k in 1:K){
			nk[k]<-sum(S_vig[-i]==k)
			prob[k]<-nk[k]*dmvnorm(Y[which(indiv==i),],mean=c(matrix(X[which(indiv==i),],ncol=ncol(X))%*%matrix(gama_vig,ncol=1)+Z[which(indiv==i),]%*%matrix(b_vig[,k],ncol=1)),sigma=diag(sigma2_vig,repl[i]))}
		prob<-c(prob,alpha_vig*dmvnorm(Y[which(indiv==i),],mean=c(matrix(X[which(indiv==i),],ncol=ncol(X))%*%matrix(gama_vig,ncol=1)+Z[which(indiv==i),]%*%beta_vig),sigma=(Z[which(indiv==i),]%*%Dmat%*%t(Z[which(indiv==i),])+diag(sigma2_vig,repl[i]))))
		S_old<-S_vig[i]
		S_vig[i]<-rDiscreta(prob/sum(prob))
		if (S_vig[i]!=S_old){
			S_obs_vig[which(indiv==i)]<-S_vig[i]
			b_vig_teste<-matrix(0,nrow=qcov,ncol=max(K,max(S_vig)))
			b_vig_teste[,1:ncol(b_vig)]<-b_vig
			b_vig<-b_vig_teste
			for (k in 1:max(K,max(S_vig))){
				if (length(which(S_obs_vig==k))>0){
					Bk<-Z[which(S_obs_vig==k),]
					aux<-solve((t(Bk)%*%Bk)+(solve(Dmat)*sigma2_vig))
					media<-aux%*%((t(Bk)%*%(Y[which(S_obs_vig==k),]-matrix(X[which(S_obs_vig==k),],ncol=ncol(X))%*%matrix(gama_vig,ncol=1)))+(sigma2_vig*solve(Dmat)%*%beta_vig))
					varcov<-aux*sigma2_vig
					b_vig[,k]<-t(rmvnorm(1, mean = c(media), sigma = varcov, method=c("chol"), pre0.9_9994 = TRUE))}}}
		while (length(table(S_vig))<max(S_vig)){ # exclude empty clusters
			categr<-as.numeric(as.character(data.frame(table(S_vig))[,1]))
			categd<-seq(1:length(table(S_vig)))
			dif<-which(categr!=categd)
			S_vig[which(S_vig>dif[1])]<-S_vig[which(S_vig>dif[1])]-1
			b_vig<-matrix(c(b_vig[,-dif[1]]),nrow=qcov)
			K<-ncol(b_vig)
			S_obs_vig<-NULL
			for (ret in 1:m) S_obs_vig<-c(S_obs_vig,rep(S_vig[ret],repl[ret]))}	
		if (length(table(S_vig))<K){
			b_vig<-matrix(c(b_vig[,-K]),nrow=qcov)
			K<-ncol(b_vig)}
		K<-length(table(S_vig))}		
	#
    ###### ordena os grupos do maior para o menor
    #
    ord=as.numeric(names(sort(table(S_vig),decreasing = TRUE)))
  	Ss=S_vig
  	Ss_obs<-S_obs_vig
  	for(l in 1:length(ord)){
  		S_vig[Ss==ord[l]]=l
  		S_obs_vig[Ss_obs==ord[l]]=l}
	b_vig<-matrix(b_vig[,ord],ncol=ncol(b_vig))
    #
	###### update matrix D
	#
	Spost<-matrix(0,ncol=ncol(sigma),nrow=nrow(sigma))
	for (k in 1:K) Spost<-((matrix(b_vig[,k],ncol=1)-beta_vig)%*%t(b_vig[,k]-beta_vig))*sum(S_vig==k)+Spost
	Dmat<-riwish(ni+m,sigma+Spost)
	#
	###### update random effects
	#
	for (k in 1:K){
		Bk<-Z[which(S_obs_vig==k),]
		aux<-solve((t(Bk)%*%Bk)+(solve(Dmat)*sigma2_vig))
		media<-aux%*%((t(Bk)%*%(Y[which(S_obs_vig==k),]-matrix(X[which(S_obs_vig==k),],ncol=ncol(X))%*%matrix(gama_vig,ncol=1)))+(sigma2_vig*solve(Dmat)%*%beta_vig))
		varcov<-aux*sigma2_vig
		b_vig[,k]<-t(rmvnorm(1, mean = c(media), sigma = varcov, method=c("chol"), pre0.9_9994 = TRUE))}		
	#
	###### update fixed effects, variance and alpha
	#
	ran_pred<-NULL
	for (i in 1:n) ran_pred[i]<-Z[i,]%*%matrix(b_vig[,S_obs_vig[i]],ncol=1)
	media_beta<-matrix(0,ncol=1,nrow=nrow(b_vig))
	for (k in 1:K) media_beta<-media_beta+(sum(S_vig==k)*b_vig[,k])
	beta_vig<-t(rmvnorm(1, mean = c(media_beta/m), sigma = Dmat/m, pre0.9_9994 = TRUE))
	residuos<-matrix(Y-ran_pred,ncol=1)
	gama_vig<-rmvnorm(1,mean=c(XXinvXtrans%*%residuos), sigma=(XXinv*sigma2_vig), pre0.9_9994 = TRUE)
	residuos<-matrix(residuos-(X%*%matrix(gama_vig,ncol=1)),ncol=1)
	sigma2_vig<-round(poster_sigma2(lambda1,lambda2,c(residuos)),7)
	alpha_vig<-gera_eta_alpha(1,a_alph_prior,b_alph_prior,K,m)
	#
	#### recording results
	#	
	if (int>burnin & int%%saltos==0){
		log_vero<-0
		for (i in 1:m) log_vero<-log_vero+dmvnorm(Y[which(indiv==i),],mean=c(matrix(X[which(indiv==i),],ncol=ncol(X))%*%matrix(gama_vig,ncol=1)+Z[which(indiv==i),]%*%matrix(b_vig[,S_vig[i]],ncol=1)),sigma=diag(sigma2_vig,repl[i]),log=TRUE)
#		
		
b.list<-list(c(b_vig))
b.sample<-est.param[["bs"]]
est.param[["bs"]] <- c(b.sample,b.list)
est.param[["Sj"]]=rbind(est.param[["Sj"]],  S_vig)
est.param[["Gamma"]]=rbind(est.param[["Gamma"]],  gama_vig)
est.param[["K"]]=rbind(est.param[["K"]],  K)
est.param[["sigma2"]]=rbind(est.param[["sigma2"]],  sigma2_vig)
est.param[["alpha"]]=rbind(est.param[["alpha"]],  alpha_vig)
est.param[["Betas"]]=rbind(est.param[["Betas"]],  beta_vig)
est.param[["matrix_D"]]=rbind(est.param[["matrix_D"]], round(Dmat,3))
est.param[["log_vero"]]=rbind(est.param[["log_vero"]],  log_vero) 
}}
```




