# ks
/*VACLASS*/
/*使用变量聚类:主成分分析法*/
proc varclus data=imputed 
             maxeigen=.7
             outtree=fortree 
             short;
   var &inputs brclus1-brclus4 miacctag 
       miphone mipos miposamt miinv 
       miinvbal micc miccbal miccpurc
       miincome mihmown milores mihmval 
       miage micrscor;
run;


ods trace on/listing;

proc varclus data=imputed 
             maxeigen=.7
             outtree=fortree 
             short;
   var &inputs brclus1-brclus4 miacctag 
       miphone mipos miposamt miinv 
       miinvbal micc miccbal miccpurc
       miincome mihmown milores mihmval 
       miage micrscor;
run;

ods trace off;

ods listing close;
ods output clusterquality=summary
           rsquare=clusters;

proc varclus data=imputed 
             maxeigen=.7 
             short 
             hi;
   var &inputs brclus1-brclus4 miacctag 
       miphone mipos miposamt miinv 
       miinvbal micc miccbal miccpurc
       miincome mihmown milores mihmval 
       miage micrscor;
run;
ods listing;


data _null_;
   set summary;
   call symput('nvar',compress(NumberOfClusters));
run;

data clusters_&nvar;
	set clusters;
	where NumberOfClusters=&nvar;
run;

proc print data=clusters;
   where NumberOfClusters=&nvar;
   id cluster;
   var Variable RSquareRatio VariableLabel;
run;


axis1 value=(font = tahoma rotate=0 height=.8) 
      label=(font = tahoma angle=90 height=2);
axis2 order=(0 to 6 by 2);

proc tree data=fortree 
          horizontal 
          vaxis=axis1 
          haxis=axis2;
   height _MAXEIG_;
run;

/*分箱*/

%MACRO Num_Var_Binning
		(	Input_DSN=, 							 	/*input data set name*/
			Target =  , 										/*target binary variables*/
			InputVar_List = , 							/*input continuous variables list, separated by blanks*/			
			Max_Missing_Portion =0.1 , 			/* how big of the  portiton of missing obs out of total obs is allowed; 0~1*/
			Initial_Bin_Num = ,						/*Inital bin number, recommeded:100 or 50 or 20 or 16 or 8*/
			Inital_Bin_Method = , 					/* Equal Width(Bucket) or Equal Height(quantitle).  1: Equal Width; 2: Equal Height*/
			Max_Bin_Num = , 							/*the maximum number of bins are allowed to form, 10~15*/
			ChiSquare_Sign_Level = ,				/* signicance level for the independence chi-square test */
			Info_Value_Threshold =, 				/*  information threshould for variable selection */
			Output_DSN_Detail =,					/*Output dataset name, detailed info   */
			Output_DSN_Smry =,					/*Output dataset name, summary info  */
			Output_DSN_Sel_Detail=,
			Output_DSN_Sel_Smry =,				/*Output dataset name, summary info */
			output_missing_var=					/*Output missing_var */
		);

	*0. dataset clean  ---add by jby on 20150826;
		proc delete data=&Output_DSN_Detail;run;
		proc delete data=&Output_DSN_Smry;run;
		proc delete data=&Output_DSN_Sel_Detail;run;
		proc delete data=&Output_DSN_Sel_Smry;run;
		proc delete data=&output_missing_var;run;
		proc delete data=temp_missing_var;run;


	*1.target variable check - missing value, boolean 0-1 ;
		PROC SQL NOPRINT;
				SELECT COUNT(*) INTO :MissTargetCnt FROM &Input_DSN WHERE &Target IS MISSING;
				SELECT MIN(&Target), MAX(&Target) INTO :MinTargetValue, :MaxTargetValue FROM &Input_DSN;
		QUIT;
		%LET MissTargetCnt = &MissTargetCnt;
		%LET MinTargetValue = &MinTargetValue;
		%LET MaxTargetValue = &MaxTargetValue;
		
		%IF &MissTargetCnt>0 OR &MinTargetValue NE 0 OR &MaxTargetValue NE 1 %THEN
			%DO;
					%PUT target variable(&Target) is not binary type or have missing values, please check;
					%RETURN;
			%END;
	
	*2.retrieve the input variable list one by one;
		%LET i =1;
		%DO %WHILE (%SCAN(&InputVar_List, &i) NE );
				%GLOBAL InputVar&i;
				%LET InputVar&i = %SCAN(&InputVar_List, &i);
				%LET i = %EVAL(&i+ 1);		
		%END;
		%GLOBAL InputVarCnt;
		%LET InputVarCnt = %EVAL(&i - 1);


	*total observations;
	%GLOBAL All_Obs_Cnt;
	DATA _NULL_;
			SET &Input_DSN(KEEP=&Target) NOBS=temp_x;
			IF _N_ = 1 THEN CALL SYMPUT("All_Obs_Cnt", TRIM(LEFT(temp_x)));
	RUN;		
	%LET All_Obs_Cnt = &All_Obs_Cnt;
	
			
	*3. binning the Input variable list one by one;
	%LET k=1;
	%DO k=1 %TO &InputVarCnt;
			PROC SQL NOPRINT;
					*Missing Value check;	
					%LET MissXvar_Cnt = 0;					
										
					*count missing values;
					SELECT COUNT(*) INTO :MissXvar_Cnt FROM &Input_DSN WHERE &&InputVar&k IS MISSING;				
					%LET MissXvar_Cnt =&MissXvar_Cnt;					
					
					*check whether the missing input observation > allowed missing part;
					%IF %SYSEVALF(&MissXvar_Cnt/&All_Obs_Cnt)>&Max_Missing_Portion %THEN %DO;																										
							%PUT ERROR:  Input Variable  &&InputVar&k  has missing value (&MissXvar_Cnt observations, accounting for %SUBSTR(%SYSEVALF(100*&MissXvar_Cnt/&All_Obs_Cnt),1,2)% of total population);
/*							%RETURN;*/

			/*save the missing var	---add by jby on 20150826*/
							data temp_missing_var;
							length missing_var $100.;
							missing_var="&&InputVar&k";
							missing_obs=&MissXvar_Cnt;
							tot_obs=&All_Obs_Cnt;
							missing_percentage=&MissXvar_Cnt/&All_Obs_Cnt;
							Max_Missing_Portion=&Max_Missing_Portion;
							run;

							%if %SYSFUNC(exist(&output_missing_var))=1 %then %do;
							proc append base=&output_missing_var  data=tmp; run; 
							%end;
							%else %do;
							data &output_missing_var;
							set temp_missing_var;
							run;
							%end;


					%END;								
					
					*although the missing part is within the allowed range, it needs to put the warning to the log;
					%IF &MissXvar_Cnt>0 %THEN %DO;
							%PUT WARNING:  Input Variable  &&InputVar&k  has missing value (&MissXvar_Cnt observations, accounting for %SUBSTR(%SYSEVALF(100*&MissXvar_Cnt/&All_Obs_Cnt),1,2)% of total population);
					%END;
			QUIT;
			
			*initial binning generation;					
			%IF &Inital_Bin_Method = 1 %THEN %DO;  /*equal width, missing part is a special width*/
					*Maximum & Minimum value for each of the X variable;
					%LET Var_X_Max = ;
					%LET Var_X_Min = ;	
					PROC SQL NOPRINT;
							SELECT MIN(&&InputVar&k), MAX(&&InputVar&k) INTO :Var_X_Min, :Var_X_Max FROM &Input_DSN;  /*excluding missing observations*/								
					QUIT;
					%LET Var_X_Max=&Var_X_Max;
					%LET Var_X_Min=&Var_X_Min;
					
					*with missing observations;
					%IF  &MissXvar_Cnt>0 %THEN %DO;
							*missing observation will be taken as an initial bin;
							*calculate the "width" for the intial binning;
							%LET Init_Width=%SYSEVALF((&Var_X_Max-&Var_X_Min)/(&Initial_Bin_Num-1));  /* initial_bin_num -1, deducting the "missing" bin;*/
							
							DATA temp_ds_bin;
									SET &Input_DSN(KEEP=&Target &&InputVar&k);
									IF  &&InputVar&k = . THEN Bin =1;
									ELSE IF &&InputVar&k = &Var_X_Min THEN Bin =2;
									ELSE IF &&InputVar&k = &Var_X_Max THEN Bin =&Initial_Bin_Num;
									ELSE	Bin =CEIL((&&InputVar&k -&Var_X_Min)/&Init_Width)+1;
							RUN;
					%END;
					
					*without missing observations;
					%IF  &MissXvar_Cnt=0 %THEN %DO;
							%LET Init_Width=%SYSEVALF((&Var_X_Max-&Var_X_Min)/(&Initial_Bin_Num));  						
							DATA temp_ds_bin;
									SET &Input_DSN(KEEP=&Target &&InputVar&k);
									IF &&InputVar&k = &Var_X_Min THEN Bin =1;
									ELSE IF &&InputVar&k = &Var_X_Max THEN Bin =&Initial_Bin_Num;
									ELSE Bin =CEIL((&&InputVar&k - &Var_X_Min)/&Init_Width);
							RUN;
					%END;						
			%END;
			
			%IF &Inital_Bin_Method = 2 %THEN %DO;  /*equal height, missing part is a special height*/
					%IF  &MissXvar_Cnt>0 %THEN %DO;
							PROC RANK DATA=&Input_DSN(KEEP=&Target &&InputVar&k) GROUPS=%EVAL(&Initial_Bin_Num-1) OUT=temp_ds_bin;
									VAR &&InputVar&k;
									RANKS Bin;
							RUN;
							
							DATA temp_ds_bin;
									SET temp_ds_bin;
									IF Bin = . THEN Bin=1;
									ELSE Bin = Bin+2;
							RUN;
					%END;
					
					%IF  &MissXvar_Cnt=0 %THEN %DO;
							PROC RANK DATA=&Input_DSN(KEEP=&Target  &&InputVar&k) GROUPS=&Initial_Bin_Num OUT=temp_ds_bin;
									VAR &&InputVar&k;
									RANKS Bin;
							RUN;
							
							DATA temp_ds_bin;
									SET temp_ds_bin;
									Bin=Bin+1;									
							RUN;
					%END;							
			%END;
			
			*get the range for each of the bin;
			PROC SQL NOPRINT;
					CREATE TABLE temp_bin_limits AS
					SELECT Bin, MIN(&&InputVar&k) AS Bin_LowerLimit, MAX(&&InputVar&k) AS Bin_UpperLimit 
						FROM temp_ds_bin 
						GROUP BY Bin
						ORDER BY Bin;						
			QUIT;
			
			*calculate the 'Event' and 'NonEvent' count for each of the bin;
			PROC SORT DATA=temp_ds_bin(KEEP=&Target Bin) OUT=temp_bin_smry;
					BY Bin;
			RUN;
			
			DATA temp_bin_smry;
					SET temp_bin_smry;
					BY Bin;
					
					*initialize;
					IF FIRST.Bin THEN DO;
							Total_Cnt = .;
							Event_Cnt = .;			/* Event:Target=1*/
							NonEvent_Cnt = .;		/*NonEvent:Targe=0*/				
					END;
					
					Total_Cnt +1;
					Event_Cnt +(&Target =1);
					NonEvent_Cnt +(&Target =0);
					
					IF LAST.Bin;
					DROP &Target;
			RUN;			
			
		*merge the min/max with the cross tabulation;					
		DATA temp_bin_smry;
			MERGE temp_bin_smry
						temp_bin_limits;
			BY Bin;
		RUN;
		
		*calculate the total Events and Nonevents;
		%LET Event_Total = ;
		%LET NonEvent_Total =  ;
		PROC SQL NOPRINT;
				SELECT SUM(Event_Cnt),  SUM(NonEvent_Cnt) INTO :Event_Total,  :NonEvent_Total FROM temp_bin_smry;
		QUIT;
		%LET Event_Total = &Event_Total;
		%LET NonEvent_Total = &NonEvent_Total;		
		
		*in case of ZERO COUNT event or ZERO COUNT nonevent;
		*collapsing in ZERO COUNT case;
		*make sure the last observation has non-zero count for both event and nonevent;
		%LET Precondition =1;
		%DO %WHILE (&Precondition=1);
				PROC SQL NOPRINT;
						SELECT MAX(Bin) INTO :LastObsIndex FROM temp_bin_smry;																		
						%LET LastObsIndex = &LastObsIndex;
						
						SELECT Total_Cnt, Event_Cnt, NonEvent_Cnt,  Bin_UpperLimit INTO: Last_Total, : Last_Event, : Last_NonEvent, :Last_Upper FROM temp_bin_smry WHERE Bin=&LastObsIndex;
						%LET Last_Total = &Last_Total;
						%LET Last_Event = &Last_Event;
						%LET Last_NonEvent = &Last_NonEvent;
						%LET Last_Upper = &Last_Upper;
						
						%IF &Last_Event <5 OR &Last_NonEvent <5  %THEN %DO;
								UPDATE temp_bin_smry
										SET
												Total_Cnt = Total_Cnt + &Last_Total,
												Event_Cnt= Event_Cnt + &Last_Event ,
												NonEvent_Cnt = NonEvent_Cnt + &Last_NonEvent,
												Bin_UpperLimit = &Last_Upper
										WHERE Bin=%EVAL(&LastObsIndex-1);
								
								DELETE  FROM temp_bin_smry WHERE Bin=&LastObsIndex;
						%END;
						
						%IF &Last_Event >= 5 AND &Last_NonEvent >= 5 %THEN %DO;
								%LET Precondition =0;
						%END;
				QUIT;
		%END;	
							
		
		DATA temp_bin_smry;
				SET temp_bin_smry;
				BY Bin;
				
				RETAIN Total Event 0 NonEvent 0 Lower 0  Upper 0  New_Start_Ind 0;
				
				*collapse;
				Total_Cnt = SUM(Total, Total_Cnt);
				Event_Cnt = SUM(Event, Event_Cnt);
				NonEvent_Cnt = SUM(NonEvent, NonEvent_Cnt);
				
				*range initialization;
				IF _N_ =1 OR New_Start_Ind=1 THEN DO;
						Lower=Bin_LowerLimit;
						Upper=Bin_UpperLimit;		
				END;
							
				IF	Event_Cnt<5 OR NonEvent_Cnt<5 THEN DO;
						Total = Total_Cnt;
						Event = Event_Cnt;
						NonEvent = NonEvent_Cnt;		
						New_Start_Ind =0;
						Lower=MIN(Bin_LowerLimit, Lower);
						Upper=MAX(Bin_UpperLimit, Upper);											
				END;
				ELSE DO;
						Bin_LowerLimit = MIN(Lower, Bin_LowerLimit);
						Bin_UpperLimit = MAX(Upper, Bin_UpperLimit);
						Total=0;
						Event=0;
						NonEvent=0;
						Lower=0;
						Upper=0;
						New_Start_Ind =1;				
						OUTPUT;
				END;
				
				DROP Total Event NonEvent Lower Upper New_Start_Ind;
		RUN;
		
		*final preparation;
		DATA temp_bin_smry;
	 		SET temp_bin_smry;
	 		Bin_Index = _N_;
		RUN;		
		
			
		*binning algorithms -- chi merge;		
		%LET ChiSquare_threshold = %SYSFUNC(QUANTILE(CHISQUARE, %SYSEVALF(1-&ChiSquare_Sign_Level),1));
		
		%LET Split_Stop_Ind = 0;  /*two conditions: the maximum bins allowed and minimum chi-squared allowed;*/
		%DO %UNTIL (&Split_Stop_Ind = 1);							
				*retrieve the current number of bins;
				%LET Current_Bin_Nr = ;
				PROC SQL NOPRINT;
						SELECT MAX(Bin_Index) INTO :Current_Bin_Nr FROM temp_bin_smry;
				QUIT;
				%LET Current_Bin_Nr = &Current_Bin_Nr;
				
				*store the minimu chisquare value and associated index to split;
				%LET BestIndex_ToSplit =0 ;
				%LET BestChiSqure_Value =10000 ;  /*10000 is an arbitrary big number*/
				
				*calculate the chi-squre statistics between each of the adjacent pairs of bins;
				%LET i=1;
				%DO i=1 %TO %EVAL(&Current_Bin_Nr - 1);			
						*count the event/nonevent/total of each pair of adjacent bins;			
						PROC SQL NOPRINT;
								SELECT SUM(Event_Cnt) , SUM(NonEvent_Cnt), SUM(Total_Cnt) INTO :Event_1, :NonEvent_1, :Total_1 FROM temp_bin_smry WHERE Bin_Index = &i;
								SELECT SUM(Event_Cnt) , SUM(NonEvent_Cnt), SUM(Total_Cnt) INTO :Event_2, :NonEvent_2, :Total_2 FROM temp_bin_smry WHERE Bin_Index = %EVAL(&i+1);
						QUIT;							
						
						*remove the leading/trailing blanks;							
						%LET Event_1 = &Event_1;
						%LET NonEvent_1 = &NonEvent_1;
						%LET Total_1 = &Total_1;
						%LET Event_2 = &Event_2;
						%LET NonEvent_2 = &NonEvent_2;
						%LET Total_2 = &Total_2;						
						
						*calculate the totals;
						%LET Total_Event = %EVAL(&Event_1 + &Event_2);
						%LET Total_NonEvent = %EVAL(&NonEvent_1 + &NonEvent_2);						
						%LET Total_All = %EVAL(&Total_Event + &Total_NonEvent);
						
						*calculate the ChiSquare Value;
						%LET ChiSquare_&i = %SYSEVALF((&Event_1 - (&Total_1*&Total_Event)/&Total_All)**2/(&Total_1*&Total_Event/&Total_All) + (&NonEvent_1 - (&Total_1*&Total_NonEvent/&Total_All))**2/(&Total_1*&Total_NonEvent/&Total_All) +
																		   (&Event_2 - (&Total_2*&Total_Event)/&Total_All)**2/(&Total_2*&Total_Event/&Total_All) + (&NonEvent_2 - (&Total_2*&Total_NonEvent/&Total_All))**2/(&Total_2*&Total_NonEvent/&Total_All));
						
						*%PUT Event_1 = &Event_1  NonEvent_1 = &NonEvent_1  Event_2 = &Event_2  NonEvent_2 = &NonEvent_2  ChiSquare = &&ChiSquare_&i;				
						%IF %SYSEVALF(&BestChiSqure_Value > &&ChiSquare_&i) %THEN %DO;
								%LET BestChiSqure_Value = &&ChiSquare_&i;
								%LET BestIndex_ToSplit=&i;
						%END;
				%END;
				
				*merge the pair with the lowest ChiSquare value;
				%IF %SYSEVALF(&BestChiSqure_Value<= &ChiSquare_threshold) OR %SYSEVALF(&Current_Bin_Nr>&Max_Bin_Num) %THEN %DO;
						PROC SQL NOPRINT;
								UPDATE temp_bin_smry
								SET Bin_Index = Bin_Index - 1
								WHERE Bin_Index > &BestIndex_ToSplit;																
						QUIT;													
				%END;
				
				*proceed to split or not;
				%IF %SYSEVALF(&BestChiSqure_Value> &ChiSquare_threshold) AND %SYSEVALF(&Current_Bin_Nr<=&Max_Bin_Num) %THEN %DO;
						%LET Split_Stop_Ind = 1;
				%END;
		
		%END;
	
		DATA temp_bin_smry;
				FORMAT  Input_Var $32.;
				SET temp_bin_smry;
				Input_Var = "&&InputVar&k";	
				
				FORMAT Bin_Event_Rate Event_Rate NonEvent_Rate WOE IV 8.4;						
				Bin_Event_Rate = Event_Cnt/Total_Cnt;
				Event_Rate = Event_Cnt/&Event_Total;
				NonEvent_Rate = NonEvent_Cnt/&NonEvent_Total;
				WOE = LOG((Event_Cnt/NonEvent_Cnt)/(&Event_Total/&NonEvent_Total));
				IV = (Event_Rate - NonEvent_Rate)*LOG(Event_Rate/NonEvent_Rate);										
		RUN;
	
		*binning for one variable is done, and ready for the next variable in the list;
		%IF &k=1 %THEN %DO;
				DATA &Output_DSN_Detail;					
						SET  temp_bin_smry;											
				RUN;						
		%END;
		
		%IF &K>1 	%THEN %DO;
				DATA &Output_DSN_Detail;
						SET &Output_DSN_Detail						
						temp_bin_smry;						
				RUN;
		%END;	
		
		*summarize ;
		PROC SORT DATA= &Output_DSN_Detail OUT=&Output_DSN_Smry;
				BY Input_Var Bin_Index Bin;
		RUN;
		
		DATA &Output_DSN_Smry(RENAME= (Total=Total_Cnt Event=Event_Cnt NonEvent=NonEvent_Cnt Lower=Bin_LowerLimit Upper=Bin_UpperLimit ));
				SET &Output_DSN_Smry(DROP=Bin_Event_Rate Event_Rate NonEvent_Rate WOE IV);
				BY Input_Var Bin_Index Bin;
			
				RETAIN   Lower Upper;
				IF FIRST.Bin_Index THEN DO;
						Total =0;
						Event=0;
						NonEvent=0;
						Lower=Bin_LowerLimit;
						Upper=Bin_UpperLimit;	
				END;
			
				Total +Total_Cnt;
				Event +Event_Cnt;
				NonEvent + NonEvent_Cnt;
				Lower= (Lower>< Bin_LowerLimit);
				Upper = MAX(Upper, Bin_UpperLimit );
			
				FORMAT Bin_Event_Rate Event_Rate NonEvent_Rate WOE IV 8.4;						
				Bin_Event_Rate = Event/Total;
				Event_Rate = Event/&Event_Total;
				NonEvent_Rate = NonEvent/&NonEvent_Total;
				WOE = LOG((Event/NonEvent)/(&Event_Total/&NonEvent_Total));
				IV = (Event_Rate - NonEvent_Rate)*LOG(Event_Rate/NonEvent_Rate);			
							
				IF LAST.Bin_Index;
				DROP Total_Cnt Event_Cnt  NonEvent_Cnt Bin_LowerLimit Bin_UpperLimit   Bin;
		RUN;
	
		PROC SQL NOPRINT;
				CREATE TABLE temp_iv_all AS
						SELECT Input_Var, SUM(IV) AS IV_VAR FROM &Output_DSN_Smry GROUP BY Input_Var HAVING SUM(IV)>=&Info_Value_Threshold;
		QUIT;
		
		PROC SORT DATA=temp_iv_all NODUPKEY;
				BY Input_Var;
		RUN;
			
		PROC SORT DATA=&Output_DSN_Detail;
				BY Input_Var;
		RUN;
		PROC SORT DATA=&Output_DSN_Smry;
				BY Input_Var;
		RUN;
		DATA &Output_DSN_Sel_Detail;
				MERGE &Output_DSN_Detail
							temp_iv_all(IN=a);
				BY Input_Var;
				
				IF a;
		RUN;
		
		DATA &Output_DSN_Sel_Smry;
				MERGE &Output_DSN_Smry
							temp_iv_all(IN=a);
				BY Input_Var;
				
				IF a;
		RUN;
	%END;
%MEND;
			
				
/*WOE*/
%macro woe_iv_single(indata,outdata,gbind,variable);
proc freq data=&indata noprint;
      tables &variable*&gbind/missing list outpct out=temp_1;
 run;
 proc contents data=temp_1 out=temp_2 noprint;
 run;
 data _null_;
     set temp_2;
     where name="&gbind";
     call symput("indtype",type);
  run;
  
  %if &indtype=1 %then %do;
     data temp_1;
        set temp_1;
      select (&gbind);
      when(1) &gbind.1="G";
      when(0) &gbind.1="B";
      when(2) delete;
    end;
   run;
   
   %end;
   %else %do;
       DATA temp_1;
       SET temp_1;
       SELECT(&gbind);
          WHEN("G","1") &gbind.1="G";
          WHEN("B","0") &gbind.1="B";
          WHEN("U","2") DELETE;
        end;
     run;
    %end;

    proc transpose DATA=temp_1 OUT=temp_3;
         var pct_col;
         BY &variable;
         id &gbind.1;
         run;
    DATA temp_4;
        SET temp_3(DROP=_name_ _label_);
        IF b=. THEN b=0;
        IF g=. THEN g=0;
        IF b=0 OR g=0 THEN woe=LOG(g/100+0.5)-LOG(b/100+0.5);
        ELSE woe=LOG(g/100)-LOG(b/100);
        retain iv 0;
        iv=iv+(g-b)/100*woe;
        run;
        DATA &outdata;
          SET temp_4 END=e;
          IF NOT e THEN iv=.;
          run;
       %mend;


%woe_iv_single(vip.Rowdata0317,temp1,bad_30,sex)
%woe_iv_single(vip.Rowdata0317,temp2,bad_30,account_channel)
%woe_iv_single(vip.Rowdata0317,temp3,bad_30,vest_flag)
%woe_iv_single(vip.Rowdata0317,temp4,bad_30,wallet_withdraw_flag)
%woe_iv_single(vip.Rowdata0317,temp5,bad_30,phone_verify_flag)

%woe_iv_single(vip.Rowdata0317,temp7,bad_30,vipshop_card_flag)
%woe_iv_single(vip.Rowdata0317,temp8,bad_30,phone_app_flag)
%woe_iv_single(vip.Rowdata0317,temp9,bad_30,reg_most_same_flag)
%woe_iv_single(vip.Rowdata0317,temp10,bad_30,receive_phone_count_type)
%woe_iv_single(vip.Rowdata0317,temp11,bad_30,rejuce_type_16m)

%woe_iv_single(vip.Rowdata0317,temp12,bad_30,xiaofei_16m)
%woe_iv_single(vip.Rowdata0317,temp13,bad_30,bijun_16m_type)
%woe_iv_single(vip.Rowdata0317,temp14,bad_30,return_type)

%woe_iv_single(vip.Rowdata0317,temp16,bad_30,most_pay_type_3m)
%woe_iv_single(vip.Rowdata0317,temp17,bad_30,is_cod_pay_count_3m)
%woe_iv_single(vip.Rowdata0317,temp18,bad_30,is_cc_pay_count_3m)
%woe_iv_single(vip.Rowdata0317,temp20,bad_30,credit_agv_type_3)
%woe_iv_single(vip.Rowdata0317,temp21,bad_30,user_unit_price_type)
%woe_iv_single(vip.Rowdata0317,temp22,bad_30,good_unit_price_type)
%woe_iv_single(vip.Rowdata0317,temp23,bad_30,receiver_count_type)
%woe_iv_single(vip.Rowdata0317,temp24,bad_30,is_c_type)
%woe_iv_single(vip.Rowdata0317,temp25,bad_30,l6m_send_address_num_type)
%woe_iv_single(vip.Rowdata0317,temp26,bad_30,AMT_AGV_TYPE)
%woe_iv_single(vip.Rowdata0317,temp27,bad_30,register_month_type)
%woe_iv_single(vip.Rowdata0317,temp28,bad_30,last_12q_cc_pay_count)
;

proc sql;
create table vip.rowdata1  as
select 'sex' as  name ,count(sex)  as numbers,sex as var,G,B,WOE,IV FROM temp1 
union
select 'account_channel' as  name,count(account_channel)  as numbers,account_channel as var,G,B,WOE,IV FROM temp2
union
select 'receive_phone_count_type' as  name,count(receive_phone_count_type) as numbers,receive_phone_count_type as var,G,B,WOE,IV FROM temp10
union
select'rejuce_type_16m' as  name,count(rejuce_type_16m) as numbers,rejuce_type_16m as var,G,B,WOE,IV FROM temp11
UNION
select 'xiaofei_16m' as  name,count(xiaofei_16m) as numbers,xiaofei_16m as var,G,B,WOE,IV FROM temp12
union
select'bijun_16m_type' as  name,count(bijun_16m_type) as numbers,bijun_16m_type as var,G,B,WOE,IV FROM temp13
union
select'most_pay_type' as  name,count(most_pay_type_3m) as numbers,most_pay_type_3m as var,G,B,WOE,IV FROM temp16
union
select'credit_agv_type_13m' as  name,count(credit_agv_type_3) as numbers,credit_agv_type_3 as var,G,B,WOE,IV FROM temp20
union
select'user_unit_price_type' as  name,count(user_unit_price_type) as numbers,user_unit_price_type as var,G,B,WOE,IV FROM temp21
union
select'good_unit_price_type' as  name,count(good_unit_price_type) as numbers,good_unit_price_type as var,G,B,WOE,IV FROM temp22
union
select'receiver_count_type' as  name,count(receiver_count_type) as numbers,receiver_count_type as var,G,B,WOE,IV FROM temp23
union
select'l6m_send_address_num_type' as  name,count(l6m_send_address_num_type) as numbers,l6m_send_address_num_type as var,G,B,WOE,IV FROM temp25
union
select'AMT_AGV_TYPE' as  name,count(AMT_AGV_TYPE) as numbers,AMT_AGV_TYPE as var,G,B,WOE,IV FROM temp26
union
select'register_month_type' as  name,count(register_month_type) as numbers,register_month_type as var,G,B,WOE,IV FROM temp27
union
select'last_12q_cc_pay_count' as  name,count(last_12q_cc_pay_count) as numbers,last_12q_cc_pay_count as var,G,B,WOE,IV FROM temp28
;
QUIT;

proc sql;
create table vip.rowdata2  as
select 'vest_flag' as  name,count(vest_flag)  as numbers,vest_flag as var,G,B,WOE,IV FROM temp3
union
select 'wallet_withdraw_flag' as  name,count(wallet_withdraw_flag) as numbers,wallet_withdraw_flag as var,G,B,WOE,IV FROM temp4
union
select'phone_verify_flag' as  name,count(phone_verify_flag) as numbers,phone_verify_flag as var,G,B,WOE,IV FROM temp5
union
select'card_verify_flag' as  name,count(card_verify_flag) as numbers,card_verify_flag as var,G,B,WOE,IV FROM temp6
union
select'vipshop_card_flag' as  name,count(vipshop_card_flag) as numbers,vipshop_card_flag as var,G,B,WOE,IV FROM temp7
union
select'phone_app_flag' as  name,count(phone_app_flag) as numbers,phone_app_flag as var,G,B,WOE,IV FROM temp8
union
select'reg_most_same_flag' as  name,count(reg_most_same_flag) as numbers,reg_most_same_flag as var,G,B,WOE,IV FROM temp9

union
select'return_type' as  name,count(return_type) as numbers,return_type as var,G,B,WOE,IV FROM temp14
union
select'pay_type_num' as  name,count(pay_type_num) as numbers,pay_type_num as var,G,B,WOE,IV FROM temp15
union
select'is_cod_pay_count' as  name,count(is_cod_pay_count_3m) as numbers,is_cod_pay_count_3m as var,G,B,WOE,IV FROM temp17
union
select'is_cc_pay_count' as  name,count(is_cc_pay_count_3m) as numbers,is_cc_pay_count_3m as var,G,B,WOE,IV FROM temp18
union
select'credit_agv_13m' as  name,count(credit_agv_13m) as numbers,credit_agv_13m as var,G,B,WOE,IV FROM temp19
union
select'is_c_type' as  name,count(is_c_type) as numbers,is_c_type as var,G,B,WOE,IV FROM temp24
;
QUIT;

/*变量转换*/
proc sql;
create table vip.rowdata3  as
select user_id,bad_30
     ,i2.woe as sexw
	 ,i1.sex
	 ,i3.woe as account_channelw
	 ,i1.account_channel
	 ,i4.woe as vest_flagw
	 ,i1.vest_flag
	 ,i5.woe as wallet_withdraw_flagw
	 ,i1.wallet_withdraw_flag
	 ,i6.woe as phone_verify_flagw
	 ,i1.phone_verify_flag
	 ,i8.woe as vipshop_card_flagw
	 ,i1.vipshop_card_flag
	 ,i9.woe as phone_app_flagw
	 ,i1.phone_app_flag
	 ,i10.woe as reg_most_same_flagw
	 ,i1.reg_most_same_flag
	 ,i11.woe as receive_phone_count_typew
	 ,i1.receive_phone_count_type
	 ,i12.woe as rejuce_type_16mw
	 ,i1.rejuce_type_16m
	 ,i13.woe as xiaofei_16mw
	 ,i1.xiaofei_16m
	 ,i14.woe as bijun_16m_typew
	 ,i1.bijun_16m_type
     ,i15.woe as return_typew
	 ,i1.return_type
	 ,i17.woe as most_pay_type_3mw
	 ,i1.most_pay_type_3m
	 ,i18.woe as is_cod_pay_count_3mw
	 ,i1.is_cod_pay_count_3m
	 ,i19.woe as is_cc_pay_countw
	 ,i1.is_cc_pay_count_3m
	 ,i21.woe as credit_agv_type_3w
	 ,i1.credit_agv_type_3
	 ,i22.woe as user_unit_price_typew
	 ,i1.user_unit_price_type
	 ,i23.woe as good_unit_price_typew
	 ,i1.good_unit_price_type
	 ,i24.woe as receiver_count_typew
	 ,i1.receiver_count_type
	 ,i25.woe as is_c_typew
	 ,i1.is_c_type
	 ,i26.woe as l6m_send_address_num_typew
	 ,i1.l6m_send_address_num_type
	 ,i27.woe as AMT_AGV_TYPEw
	 ,i1.AMT_AGV_TYPE
	 ,i28.woe as register_month_typew
	 ,i1.register_month_type
	 ,i29.woe as last_12q_cc_pay_countw
	 ,i1.last_12q_cc_pay_count
from vip.Rowdata0317 i1
left join temp1 i2
on i1.sex=i2.sex
left join temp2 i3
on i1.account_channel=i3.account_channel
left join temp3 i4
on i1.vest_flag=i4.vest_flag
left join temp4 i5
on i1.wallet_withdraw_flag=i5.wallet_withdraw_flag
left join temp5 i6
on i1.phone_verify_flag=i6.phone_verify_flag
left join temp7 i8
on i1.vipshop_card_flag=i8.vipshop_card_flag
left join temp8 i9
on i1.phone_app_flag=i9.phone_app_flag
left join temp9 i10
on i1.reg_most_same_flag=i10.reg_most_same_flag
left join temp10 i11
on i1.receive_phone_count_type=i11.receive_phone_count_type
left join temp11 i12
on i1.rejuce_type_16m=i12.rejuce_type_16m
left join temp12 i13
on i1.xiaofei_16m=i13.xiaofei_16m
left join temp13 i14
on i1.bijun_16m_type=i14.bijun_16m_type
left join temp14 i15
on i1.return_type=i15.return_type
left join temp16 i17
on i1.most_pay_type_3m=i17.most_pay_type_3m
left join temp17 i18
on i1.is_cod_pay_count_3m=i18.is_cod_pay_count_3m
left join temp18 i19
on i1.is_cc_pay_count_3m=i19.is_cc_pay_count_3m
left join temp20 i21
on i1.credit_agv_type_3=i21.credit_agv_type_3
left join temp21 i22
on i1.user_unit_price_type=i22.user_unit_price_type
left join temp22 i23
on i1.good_unit_price_type=i23.good_unit_price_type
left join temp23 i24
on i1.receiver_count_type=i24.receiver_count_type
left join temp24 i25
on i1.is_c_type=i25.is_c_type
left join temp25 i26
on i1.l6m_send_address_num_type=i26.l6m_send_address_num_type
left join temp26 i27
on i1.AMT_AGV_TYPE=i27.AMT_AGV_TYPE
left join temp27 i28
on i1.register_month_type=i28.register_month_type
left join temp28 i29
on i1.last_12q_cc_pay_count=i29.last_12q_cc_pay_count
;
quit;

%let inputs= 
    sexw
    wallet_withdraw_flagw
	phone_verify_flagw
    vipshop_card_flagw
	phone_app_flagw
    reg_most_same_flagw
    receive_phone_count_typew
    rejuce_type_16mw
	xiaofei_16mw
    bijun_16m_typew
    return_typew
	most_pay_type_3mw
    is_cod_pay_count_3mw
    is_cc_pay_countw
    credit_agv_type_3w
    user_unit_price_typew
	good_unit_price_typew
    receiver_count_typew
	is_c_typew
	l6m_send_address_num_typew
    AMT_AGV_TYPEw
    register_month_typew
   last_12q_cc_pay_countw
;
	
/* 对样本数据分成训练数据和验证数据*/
proc sort data= vip.rowdata3 out= vip.rowdata3; 
   by bad_30; 
run; 

/*proc surveyselect noprint
                  data = vip.rowdata3
                  samprate=.7
                  out=develop
                  seed=44444
                  outall;
   strata bad_30 ;
run;
proc freq data = develop;
tables bad_30*selected;
run;
/*创建训练数据和验证数据 */
/*data  vip.train  vip.valid;
   set develop;
   if selected then output  vip.train;
   else output  vip.valid;
run;*/
/*选变量&跑逻辑回归系数*/
 proc logistic data = vip.rowdata3 des;
   model bad_30=&inputs 
   /selection=stepwise 
    slentry=0.05
    slstay=.05
	details
	outroc=roc;
run;


/*score facor,offset 计算*/
data vip.ratio;
factor=20/log(2);
offset=600-factor*log(50);
run;
/* alipay_score_typet
hours_type
last_dpd_type
type
*/

/*score 转换*/
data vip.rowdata3_1;
set vip.rowdata3   ;
     if user_unit_price_type="01:[<=100]"   then user_unit_price_type_s=69;
else if user_unit_price_type="02:[<=300]"   then user_unit_price_type_s=75;
else if user_unit_price_type="03:[<=500]"   then user_unit_price_type_s=75;
else if user_unit_price_type="04:[<=1000]"   then user_unit_price_type_s=76;
else if user_unit_price_type="05:[>1000]"   then user_unit_price_type_s=53;

if sex="F"  then sex_s=83;
else if  sex="M"  then sex_s=61;

  if register_month_type="<=6" then register_month_type_s=70;
 else if register_month_type="<=24" then register_month_type_s=73;
 else if register_month_type="<=36" then register_month_type_s=88;
 else if register_month_type=">36" then register_month_type_s=99;

 if most_pay_type_3m='0'   then most_pay_type_s=54;
 else if most_pay_type_3m='cod'   then most_pay_type_s=56;
else if most_pay_type_3m='cc'   then most_pay_type_s=85;
else if most_pay_type_3m='dc'   then most_pay_type_s=92;
else if most_pay_type_3m='vip'   then most_pay_type_s=78;

 if l6m_send_address_num_type='0'  then l6m_send_address_num_type_s=47;
else if l6m_send_address_num_type='1'   then l6m_send_address_num_type_s=72;
else if l6m_send_address_num_type='2'   then l6m_send_address_num_type_s=77;
else if l6m_send_address_num_type='[>2]'   then l6m_send_address_num_type_s=86;


 if good_unit_price_type='01:[<=100]'      then good_unit_price_type_s=76;
else if good_unit_price_type='02:[<=300]'  then good_unit_price_type_s=74;
else if good_unit_price_type='03:[<=500]'  then good_unit_price_type_s=72;
else if good_unit_price_type='04:[<=1000'  then good_unit_price_type_s=68;
else if good_unit_price_type='05:[>1000]'  then good_unit_price_type_s=36;

 if rejuce_type_16m='0'        then rejuce_type_16m_s=74;
else if rejuce_type_16m='1'    then rejuce_type_16m_s=61;
else if rejuce_type_16m='>=2'  then rejuce_type_16m_s=51;

if credit_agv_type_3='0'                then credit_agv_type_13m_s=71;
else if credit_agv_type_3='01:[<=300]'  then credit_agv_type_13m_s=80;
else if credit_agv_type_3='02:[<=500]'  then credit_agv_type_13m_s=85;
else if credit_agv_type_3='03:[<=1000]'  then credit_agv_type_13m_s=74;
else if credit_agv_type_3='04：[>1000]'  then credit_agv_type_13m_s=69;
run;

data vip.rowdata3_2;
set vip.rowdata3_1;
score=user_unit_price_type_s+sex_s+register_month_type_s+most_pay_type_s+l6m_send_address_num_type_s+good_unit_price_type_s+rejuce_type_16m_s+credit_agv_type_13m_s;
run;


%KSStat(DSin=vip.rowdata3_2, ProbVar=score,DVVar=bad_30, DSKS=vip.old_ks_a, M_KS=m_ks);

%MACRO KSStat(DSin, ProbVar, DVVar, DSKS, M_KS);
/* Calculation of the KS Statistic from the results of  a predictive model. DSin is the dataset with a dependent  variable DVVar, and a predicted probability ProbVar. 
   The KS statistic is returnd in the parameter M_KS. DSKS contains the data of the Lorenz curve for good and bad as well as the KS Curve. 
*/

	/* Sort the observations using the predicted probability */
	PROC SORT DATA = &DSin;
			BY &ProbVar;
	RUN;


	/* Find the total number of positives and negatives */
	PROC SQL NOPRINT;
			SELECT SUM(&DVVar), COUNT(*) INTO :Positive,  :Total FROM &DSin;			
	QUIT;
	%LET Positive = &Positive;
	%LET Total = &Total;
	%LET Negative = %EVAL(&Total - &Positive);
 

 	/* The base of calculation is percentile */
	/* Count number of positive and negatives for each percentile, their proportions and cumulative proportions */
	DATA &DSKS;
			SET &Dsin NOBS=Total_Cnt;
			BY &ProbVar;
			
			RETAIN Group_Index 1  Group_Positive_Cnt  0 Group_Negative_Cnt 0;
			
			Group_Size=Total_Cnt/100;

			IF &DVVar=1 THEN Group_Positive_Cnt+1;
			ELSE Group_Negative_Cnt +1;

			/* end of tile? */
			IF _N_ = CEIL(Group_Index*Group_Size) OR _N_ = Total_Cnt THEN DO;
					Group_Postive_Portion=Group_Positive_Cnt/&Positive;
					Group_Negative_Portion =Group_Negative_Cnt/&Negative;
  					OUTPUT;
   					IF Group_Index <100 THEN DO;
         						Group_Index +1;
	   				END;
  			END;	
			KEEP Group_Index   Group_Postive_Portion   Group_Negative_Portion;
	RUN;

	/* add the point of zero  */
	DATA temp_zhu_ks;
	 		Group_Index=0;
	 		Group_Postive_Portion=0;
	 		Group_Negative_Portion=0;
	RUN;

	Data &DSKS;
  			SET temp_zhu_ks &DSKS;
	RUN;

	/* Scale the tile to represent percentage and add labels*/
	DATA &DSKS;
			SET &DSKS;
			Group_Index=Group_Index/100;
			LABEL Group_Postive_Portion='Percent of Positives';
			LABEL Group_Negative_Portion ='Percent of Negatives';
			LABEL Group_Index ='Percent of population';

			/* calculate the KS Curve */
			KS=Group_Negative_Portion-Group_Postive_Portion;
	RUN;

	/* calculate the KS statistic */
	PROC SQL NOPRINT;
 			SELECT MAX(ABS(KS)) INTO :&M_KS FROM &DSKS;			
	QUIT;
	
	/* Clean the workspace */
	PROC DATASETS LIBRARY=WORK NODETAILS NOLIST;
 		DELETE temp_zhu_ks ;
	RUN;


	/* Plotting the KS curve using gplot using simple options */
	ODS GRAPHICS ON;
	symbol1 value=dot color=red   interpol=join  height=1;
 	legend1 position=top;
 	symbol2 value=dot color=blue  interpol=join  height=1;
 	symbol3 value=dot color=green interpol=join  height=1;
	PROC GPLOT DATA=&DSKS;
			TITLE "THE KS Statistic is: &M_KS";
 			PLOT( Group_Negative_Portion Group_Postive_Portion KS)*Group_Index /OVERLAY LEGEND=legend1;
 	RUN;

	GOPTIONS RESET=ALL;
	TITLE '';
	ODS GRAPHICS OFF;
%MEND;





%KSStat(DSin=vip.rowdata3_2, ProbVar=score,DVVar=bad_30, DSKS=vip.old_ks_a, M_KS=m_ks);
/*验证*/
data vip.valid_1;
set vip.valid   ;
     if xiaofei_16m="m1"   then xiaofei_16m_s=63;
else if xiaofei_16m="m2"   then xiaofei_16m_s=60;
else if xiaofei_16m="m3"   then xiaofei_16m_s=49;
else if xiaofei_16m="m4"   then xiaofei_16m_s=37;

if sex="F"  then sex_s=68;
else if  sex="M"  then sex_s=47;

  if register_month_type="<=6" then register_month_type_s=54;
 else if register_month_type="<=24" then register_month_type_s=58;
 else if register_month_type="<=36" then register_month_type_s=80;
 else if register_month_type=">36" then register_month_type_s=97;

 if most_pay_type_3m='0'   then most_pay_type_s=46;
 else if most_pay_type_3m='cod'   then most_pay_type_s=48;
else if most_pay_type_3m='cc'   then most_pay_type_s=66;
else if most_pay_type_3m='dc'   then most_pay_type_s=71;
else if most_pay_type_3m='vip'   then most_pay_type_s=62;

 if l6m_send_address_num_type='0'  then l6m_send_address_num_type_s=40;
else if l6m_send_address_num_type='1'   then l6m_send_address_num_type_s=58;
else if l6m_send_address_num_type='2'   then l6m_send_address_num_type_s=61;
else if l6m_send_address_num_type='[>2]'   then l6m_send_address_num_type_s=68;

 if is_cod_pay_count_3m='0'  then is_cc_pay_count_s=63;
else if is_cod_pay_count_3m='1'  then is_cc_pay_count_s=48;

 if good_unit_price_type='01:[<=100]'  then good_unit_price_type_s=62;
else if good_unit_price_type='02:[<=300]'  then good_unit_price_type_s=60;
else if good_unit_price_type='03:[<=500]'  then good_unit_price_type_s=57;
else if good_unit_price_type='04:[<=1000'  then good_unit_price_type_s=51;
else if good_unit_price_type='05:[>1000]'  then good_unit_price_type_s=9;

 if account_channel='html5'  then account_channel_s=64;
else if account_channel='pc'  then account_channel_s=40;

if credit_agv_type_3='0'  then credit_agv_type_13m_s=56;
else if credit_agv_type_3='01:[<=300]'  then credit_agv_type_13m_s=66;
else if credit_agv_type_3='02:[<=500]'  then credit_agv_type_13m_s=72;
else if credit_agv_type_3='03:[<=1000]'  then credit_agv_type_13m_s=60;
else if credit_agv_type_3='04：[>1000]'  then credit_agv_type_13m_s=54;
run;



data vip.valid_2;
set vip.valid_1;
score=xiaofei_16m_s+sex_s+register_month_type_s+most_pay_type_s+l6m_send_address_num_type_s+is_cc_pay_count_s+good_unit_price_type_s+account_channel_s;
run;

%KSStat(DSin=vip.valid_2, ProbVar=score,DVVar=bad_30, DSKS=vip.old_ks, M_KS=m_ks);

data vip.sroce;
set vip.valid_2 vip.train_2;
is_sucess=1;
run;
data vip.sroce;
set vip.valid_2 vip.train_2;
is_sucess=1;
run;

data vip.rowdata3_3;
set vip.rowdata3_2;
length score_type $12.;
     if score<500 then score_type="<500";
else if score<550 then score_type="<550";
else if score<575 then score_type="<575";
else if score<590 then score_type="<600";
else if score<605 then score_type="<610";
else if score<620 then score_type="<620";
else if score<635 then score_type="<630";
else if score>=635 then score_type=">=630";
run;

proc summary data=vip.rowdata3_2 n max min ;
class score;
output out=tmp1;
run;
proc print data=tmp1;
run;


proc summary data=vip.rowdata3_3 n sum  ;
class score_type bad_30 ;
output out=tmp2;
run;
proc print data=tmp2;
run;
