# utl_benchmarks_hash_merge_of_two_un-sorted_data_sets_with_some_common_variables
Benchmarks for a hash merge of two un-sorted data sets with some common variables. Keywords: sas sql join merge big data analytics macros oracle teradata mysql sas communities stackoverflow statistics artificial inteligence AI Python R Java Javascript WPS Matlab SPSS Scala Perl C C# Excel MS Access JSON graphics maps NLP natural language processing machine learning igraph DOSUBL DOW loop stackoverflow SAS community.
    Benchmarks for a hash merge of two un-sorted data sets with some common variables

    This solution was provided by: Paul Dorfman <sashole@bellsouth.net>
    This an important algorithm that deserves further analysis.

     BENCHMARKS (seconds)
     ====================

      SINGLE VANILLA SQL JOB
      =======================

      1,415  Simple SQL Join SGIO=yes bufno=100
        888  Simple SQL Join SGIO=no bufno=10


      SINGLE  HASH (Paul's algorithm)
      ================================

        345  Pauls Hash Join SGIO=yes bufno=100
        272  Pauls Hash Join SGIO=no bufno=10


      FIVE SIMULTANEOUS TASKS (USING SAS SYSTASK ONE PER YEAR)
      ========================================================

        218  Pauls Hash Join SGIO=yes bufno=100
        367  Pauls Hash Join SGIO=no bufno=10 (My experience indicates 3-4 simutaneous
                                               I/O dominated tasks is optimum.
                                               In memory cpu-intensive can benefit
                                               with much higher multi-programming levels )

        You need to combine the five outputs using a view.

    see
    https://goo.gl/aEnBde
    https://github.com/rogerjdeangelis/utl_benchmarks_hash_merge_of_two_un-sorted_data_sets_with_some_common_variables


    SAS 9.4M2 64bit and Win7 64bit. SGIO is windows only?

    I added some I/O options that allow improved  mutiprogramming.

    PROBLEM
    =======

    Superfast Hash join of a 13gb table with a 24gb table on

                               TABLES
                               ======
     _2004(23gb 68mm obs 73 cols)     C2FINA (13gb 24mm obs 45 cols)
     ----------------------------     -------------------------------
      Key                               Key
      ===                               ==

      ryear                           = year
      msa                             = msa
      countycode                      = countycode
      state                           = state
      loan_amount                     = loan_amount

    Users should try combinations of SQL and Hash with SGIO and bufno.
    I kept pagesize at 64k.
    What creates a cache miss can be very complex. Flushing the cache
    was not an issue here.

    The key to performance I/O is always having locality of
    referenece in mind to minimize cache misses. Use every
    byte in the cache taht is closest to the processor.

    HASH Algorithm

      1. Table 1: Load the 16 byte md5 hash of the concatenation of the primary keys along
         with '_n_' record number(rid) into in memory hash table.

      2. Table 2: Create the same md5 hash in the larger table. (and bring in the entire PDV)

      3. Lookup the Table 2: md5 hash(from 2.) in the in Table 1: memory hash table and
         when there is a match retrieve the record number(_n_) for Table 1.

      4. Use point=record number on Table 1 data and  retrieve the matching  record.

    INPUT  (SGIO=yes and bufno=100)
    ===============================

    libname c "c:/wrk";
    libname d "d:/wrk";

    data c.c2final(sgio=yes bufno=100 index=(ryear));

       array as[40] $8 b1-b40 (40*'12345678');
       array ns[28]    m1-m28 (28*12345);

       do ryear='2001','2002','2003','2004','2005';
          do rMSA='A','B','C','D';
             do rCountyCode='12','13','14','15','16';
                do rState='AK','AL','CA','VT','TX';
                   do rLoan_Amount=1 to 48000 by 1;
                         output;
                   end;
                end;
             end;
          end;
       end;

    run;quit;

    libname d "d:/wrk";
    data d._2004(sgio=yes bufno=100 index=(year));;
       array as[20] $8 a1-a20 (20*'12345678');
       array ns[20] n1-n20 (20*12345678);

       do year='2001','2002','2003','2004','2005';
          do MSA='A','B','C','D';
             do CountyCode='12','13','14','15','16';
                do State='AK','AL','CA','VT','TX';
                   do Loan_Amount=0 to 27200000 by 200;
                     output;
                   end;
                end;
             end;
          end;
       end;
    run;quit;


    ROCESS
    ======

      * SINGLE HASH;

       data want (drop = key) ;
         dcl hash h(hashexp:20, multidata:"y") ;
         h.definekey ("key") ;
         h.definedata ("rid") ;
         h.definedone() ;
         length key $ 16 ;
         do rid = 1 by 1 until (z1) ;
           set g.c2final ( keep=rYear--rLoan_Amount) end = z1 ;
           key = md5 (catx (":", of rYear--rLoan_Amount)) ;
           h.add() ;
         end ;
         do until (z2) ;
           set f._2004 end = z2 ;
           key = md5 (catx (":", of Year--Loan_Amount)) ;
           do _iorc_ = h.find() by 0 while (_iorc_ = 0) ;
             set g.c2final point = rid ;
             output ;
             _iorc_ = h.find_next() ;
           end ;
         end ;
         stop ;
       run;quit;


    OUTPUT
    ======

     WORK.WANT 118 variables and 120,000 observations

       Middle Observation(60000 ) of want - Total Obs 120,000

       YEAR             C 4       2003
       MSA              C 1       B
       COUNTYCODE       C 2       16
       STATE            C 2       VT
       LOAN_AMOUNT      N 8       48000

       RYEAR            C 4       2003
       RMSA             C 1       B
       RCOUNTYCODE      C 2       16
       RSTATE           C 2       VT
       RLOAN_AMOUNT     N 8       48000

        -- CHARACTER --
       A1               C 8       12345678
       A2               C 8       12345678
       A3               C 8       12345678
       ......
       B37              C 8       12345678
       B38              C 8       12345678
       B39              C 8       12345678
       B40              C 8       12345678


        -- NUMERIC --
       N1               N 8       12345678
       N2               N 8       12345678
       N3               N 8       12345678
       ......
       M26              N 8       12345678
       M27              N 8       12345678
       M28              N 8       12345678

    *                      _ _      _
     _ __   __ _ _ __ __ _| | | ___| |
    | '_ \ / _` | '__/ _` | | |/ _ \ |
    | |_) | (_| | | | (_| | | |  __/ |
    | .__/ \__,_|_|  \__,_|_|_|\___|_|
    |_|
    ;

    * Five parallel tasks. One per year. Index is important? or use firstobs obs to partition;

    %let _s=%sysfunc(compbl(C:\Progra~1\SASHome\SASFoundation\9.4\sas.exe -sysin c:\nul
      -sasautos c:\oto -autoexec c:\oto\Tut_Oto.sas -work d:\wrk));

    data _null_;file "c:\oto\utl_hshTsk.sas" lrecl=512;input;put _infile_;putlog _infile_;
    cards4;
    %macro utl_hshTsk(year);

       libname f "f:/wrk";
       libname g "g:/wrk";

       data c2final/view=c2final;
         set g.c2final(where=(ryear="&year"));
       run;quit;

       data want&year (drop = key) ;
         dcl hash h(hashexp:20, multidata:"y") ;
         h.definekey ("key") ;
         h.definedata ("rid") ;
         h.definedone() ;
         length key $ 16 ;
         do rid = 1 by 1 until (z1) ;
           set g.c2final (where=(ryear="&year") keep=rYear--rLoan_Amount) end = z1 ;
           key = md5 (catx (of rYear--rLoan_Amount)) ;
           h.add() ;
         end ;
         do until (z2) ;
           set f._2004(where=(year="&year")) end = z2 ;
           key = md5 (catx (of Year--Loan_Amount)) ;
           do _iorc_ = h.find() by 0 while (_iorc_ = 0) ;
             set c2final point = rid ;
             output ;
             _iorc_ = h.find_next() ;
           end ;
         end ;
         stop ;
       run;quit;

    %mend utl_hshTsk;
    ;;;;
    run;quit;

    * check interactively;
    %include "c:/oto/utl_hshTsk.sas";
    %utl_hshTsk(2003);

    systask kill sys1 sys2 sys3 sys4  sys5 ;
    systask command "&_s -termstmt %nrstr(%utl_hshTsk(2001);) -log d:\log\a1.log" taskname=sys1;
    systask command "&_s -termstmt %nrstr(%utl_hshTsk(2002);) -log d:\log\a2.log" taskname=sys2;
    systask command "&_s -termstmt %nrstr(%utl_hshTsk(2003);) -log d:\log\a3.log" taskname=sys3;
    systask command "&_s -termstmt %nrstr(%utl_hshTsk(2004);) -log d:\log\a4.log" taskname=sys4;
    systask command "&_s -termstmt %nrstr(%utl_hshTsk(2005);) -log d:\log\a5.log" taskname=sys5;
    waitfor sys1 sys2 sys3 sys4  sys5;



    *                  _
     _ __   __ _ _   _| |
    | '_ \ / _` | | | | |
    | |_) | (_| | |_| | |
    | .__/ \__,_|\__,_|_|
    |_|
    ;


    Roger,

    Thanks for doing that.

    Note that the crux of the matter here is using the RID instead of overloading the hash
    data portion with a slew of satellite variables and using POINT=RID downstream. Privately,
    I call it "data portion disk offloading" (first overtly proposed, IIRC, at SUGI 31 in SF).

    As far as the MD5 goes, it works really well when you have a truly long mixed-type composite
    key. In this case, with 6 numeric hash variables per hash item, it cuts the hash memory
    footprint somewhat, too, but only by 16 bytes per item (80 bytes against 64 under X64_7PRO).
    Which is why methinks it may run faster if you simply include the
    keys in the key portion and match on them, t
    hereby getting rid of the CATX+MD5 overhead.

    Best regards



