ha estimater
================
Housen Chu
April 26, 2018

Background
----------

The foundation of the *Pennypacker and Baldocchi* \[2015\] method is the logarithmic wind profile defined by Monin-Obukhov similarity theory (MOST) under near-neutral stability conditions (i.e., |(zm-d)/L| &lt; 0.1) \[*Raupach*, 1994; *Raupach*, 1995\].

***k \* WS / USTAR = ln((zm - d)/z0)+ln(lamda)***

where k = 0.4 is the von Karman constant, L is Monin-Obukhov length, USTAR is friction velocity , WS is the mean horizontal wind speed at height zm, z0 and d are roughness length and zero-plane displacement heights, respectively. MOST describes USTAR and WS above the canopy as a logarithmic function of z0 and d. ln(lamda) is an influence function associated with the roughness sublayer-a region just above the canopy where turbulence is enhanced \[*Raupach*, 1994; *Raupach*, 1995\], and lamda = 1.25 is assumed when WS is measured relatively close to the canopy top (i.e., zm &lt; 1.5hc, hc: physical canopy height) \[*Massman*, 1997; *Massman et al.*, 2017\]. Otherwise, ln(lamda) is assumed negligible (i.e., lamda = 1.00).

Both z0 and d are typically expressed as fractions of hc, where coef1 = z0 / hc and coef2 =d / hc. The above equation is then rearranged as a function of hc depending on coef1, coef2, zm, WS, and USTAR. zm is typically fixed at the sites. Given any known coef1 and coef2, this theoretical canopy height can be calculated from the measured WS and USTAR \[*Pennypacker and Baldocchi*, 2015\]. We define this theoretical height as the aerodynamic canopy height (ha), because it reflects the canopy's momentum absorption characteristics.

***k \* WS / USTAR = ln(lamda \* ((zm - coef2 \* hc)/(coef1 \* hc))***

***hc = ha = (lamda \* zm) / (lamda \* coef2 + coef1 \* exp(k \* WS / USTAR))***

### Variable list

-   **WS**: wind speed (m s-1)
-   **USTAR**: friction velocity (m s-1)
-   **k**: von karman constant, 0.4
-   **zm**: measurement height, (m)
-   **z0**: roughness length (m)
-   **d**: zero-plane displacement height (m)
-   **z**: (= zm - d) sensor height above displacement height (m)
-   **L**: Monin-Obukhov length (m)
-   **lamda**: roughness sublayer enhancing factor (1.00~1.25)

### Input arguments

-   **data.in**: R data.frame contains WS (m s-1), USTAR (m s-1), MO\_LENGTH (m), zm (m), at t consecutive time steps
-   **zmax**: maximum zm (measurement height of WS) in the data.in, this is used for the first guess of z in z/L, and for setting the acceptable upper bound for calculated ha
-   **d.hr**: N of time steps per day, default as 48 for half-hourly files, use 24 for hourly files
-   **coef1**: z0/hc, default as 0.1 if unspecified
-   **coef2**: d/hc, default as 0.6 if unspecified
-   **lamda**: roughness sublayer enhancing factor (1.00~1.25, Massman 1997, 2017), default as 1 (i.e., no RS correction)
-   **zL.cut**: cutoff z/L for near-neutral stability, i.e., zL.cut &gt; near-neutral &gt; -zL.cut, default as 0.1
-   **zm.cut**: (TRUE/FALSE) whether remove ha estimates that are higher than 1.1 \* zm, or not, default as yes

### Output values

-   **ha**: estimated aerodynamic canopy height
-   **z**: (= zm - d) sensor height above displacement height
-   **z0**: roughness length, based on ha\*coef1
-   **d**: zero-plane displacement height, based on ha\*coef2
-   **N**: Number of available data for calculation
-   **Nin**: Number of supposed data length in input

``` r
hc_ddb2<-function(data.in,         
                  zmax,
                  d.hr = 48,
                  coef1 = 0.1,
                  coef2 = 0.6,
                  lamda = 1,
                  zL.cut = 0.1,
                  zm.cut = T)
  {
  
  na.mean<-function(x) {ifelse(!is.nan(mean(x,na.rm=T)),mean(x,na.rm=T),NA)}
  na.median<-function(x) {ifelse(!is.nan(median(x,na.rm=T)),median(x,na.rm=T),NA)}
  
  k<-0.4 # von karman constant

  Nin<-nrow(data.in)
  
  # check data availability and filtering by stability range
  # set a wider range of zL for pre-run
  data.t<-data.in
  data.t[!is.na(data.t$MO_LENGTH)&zmax/data.t$MO_LENGTH<(-1),c("WS","USTAR")]<-NA  
  data.t[!is.na(data.t$MO_LENGTH)&zmax/data.t$MO_LENGTH>(1),c("WS","USTAR")]<-NA  
  
  # dont run if insufficient data, < 7 % of supposed N
  if(nrow(na.omit(data.t[,c("WS","USTAR","MO_LENGTH")]))<0.07*Nin){
    
    N<-nrow(na.omit(data.t[,c("WS","USTAR","MO_LENGTH")]))
    ha<-NA
    z<-NA
    z0<-NA
    d<-NA
    d.unc<-NA
    ha.unc<-NA
    z.unc<-NA
    z0.unc<-NA
    
  }else{
    
    # 1st run, calculate the supposed z (= zm - d) for filtering out non-near neutral conditions
    y<-data.t$zm*lamda/(coef2*lamda+coef1*exp(k*data.t$WS/data.t$USTAR));  # y: hold calculated ha for each (half)hour 
    
    if(zm.cut){
      y[y>1.1*zmax]<-NA  # set y to missing if exceeding zmax  
    }
    
    y2<-apply(matrix(y,nrow=d.hr),2,na.median)  # y2: hold calculated ha for each day
    
    ha<-na.median(y2)  # obtain best estimate from daily-estimates
    z<-ifelse(is.nan(ha),NA,na.mean(data.t$zm)-coef2*ha)
    
    rm(list=c("y","y2"))
    
    if(!is.na(z)){
      
      # 2nd run, used z (=zm-d) from 1st run to filter out non-neutral conditions 
      # a narrower range of zL for 2nd run
      data.t[!is.na(data.t$MO_LENGTH)&z/data.t$MO_LENGTH<(-zL.cut),c("WS","USTAR")]<-NA  
      data.t[!is.na(data.t$MO_LENGTH)&z/data.t$MO_LENGTH>(zL.cut),c("WS","USTAR")]<-NA  

      N<-nrow(na.omit(data.t[,c("WS","USTAR","MO_LENGTH")]))
      
      y<-data.t$zm*lamda/(coef2*lamda+coef1*exp(k*data.t$WS/data.t$USTAR));  # y: hold calculated ha for each (half)hour 
      
      if(zm.cut){
        y[y>1.1*zmax]<-NA   # set y to missing if exceeding zmax
      }
      
      y2<-apply(matrix(y,nrow=d.hr),2,na.median)  # y2: hold calculated ha for each day
      
      # best estimate from daily-estimates
      ha<-na.median(y2)  
      # uncertainty of ha, based on the deviation between best estimates ha, and the daily-estimates (not used in manuscript)  
      ha.unc<-abs(ha-y2[ceiling(length(y2)/2)])   
      
      z<-na.mean(data.t$zm)-coef2*ha
      z0<-coef1*ha
      z.unc<-coef2*ha.unc
      z0.unc<-coef1*ha.unc
      
      d<-ifelse(is.nan(z),NA,na.mean(data.t$zm)-z);
      d.unc<-z.unc
      
      rm(list=c("y","y2"))
      
    }else{
      
      N<-nrow(na.omit(data.t[,c("WS","USTAR","MO_LENGTH")]))
      ha<-NA
      ha.unc<-NA
      d<-NA
      d.unc<-NA
      z<-NA
      z0<-NA
      z.unc<-NA
      z0.unc<-NA
      
    }
  }
  return(list(ha,ha.unc,z,z.unc,z0,z0.unc,d,d.unc,N,Nin))
}
```
