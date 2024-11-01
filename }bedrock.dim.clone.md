#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.dim.clone', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pSrcDim', '', 'pTgtDim', '', 'pHier', '*',
    	'pAttr', 0, 'pUnwind', 0, 'pDelim', '&'
	);
EndIf;
#EndRegion CallThisProcess

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

#Region @DOC
# Description:
# This process can clone a source dimension and all the Hierarchies.

# Use case: Intended for development/prototyping.
# 1/ Replicate a dimension with it's hierarchies for testing.

# Note:
# If the target dimension:hierarchy already exists then it will be overwritten.
# Naturally, a valid source dimension name (pSrcDim) is mandatory otherwise the process will abort.
#EndRegion @DOC

### Global Variables
StringGlobalVariable ('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode = 0;

### Constants ###
cThisProcName     = GetProcessName();
cUserName         = TM1User();
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cSubset           = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = '%cThisProcName% : %sMessage% : %cUserName%';
cMsgInfoContent   = 'User:%cUserName% Process:%cThisProcName% Message:%sMessage%';
cLogInfo          = '***Parameters for Process:%cThisProcName% for pSrcDim:%pSrcDim%, pTgtDim:%pTgtDim%, pHier:%pHier%, pAttr:%pAttr%, pUnwind:%pUnwind%, pDelim:%pDelim%.';
cLangDim          = '}Cultures';
nNumLang          = DimSiz( cLangDim );

## LogOutput parameters
IF ( pLogoutput = 1 );
  LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

If( Scan( '*', pSrcDim )=0 &  Scan( '?', pSrcDim )=0 & Scan( pDelim, pSrcDim )=0 & Scan( ':', pSrcDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pSrcDim, Scan( ':', pSrcDim ) + 1, Long( pSrcDim ) );
    pSrcDim        = SubSt( pSrcDim, 1, Scan( ':', pSrcDim ) - 1 );
EndIf;

## Validate dimension
IF( Trim( pSrcDim ) @= '' );
  nErrors = 1;
  sMessage = 'No source dimension specified.';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIF( DimensionExists( pSrcDim ) = 0 );
  nErrors = 1;
  sMessage = 'Invalid source dimension: ' | pSrcDim;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( pTgtDim @= '' % pTgtDim @= pSrcDim );
  pTgtDim = pSrcDim | '_Clone';
EndIf;

# Validate hierarchy
IF( Scan( '*', pSrcDim )=0 &  Scan( '?', pSrcDim )=0 & Scan( pDelim, pSrcDim )=0 & pHier @= '');
    pHier = pSrcDim;
ElseIf( Scan( '*', pHier )=0 &  Scan( '?', pHier )=0 & Scan( pDelim, pHier )=0 & pHier @<> '' & HierarchyExists(pSrcDim, pHier) = 0 );
    nErrors = 1;
    sMessage = 'Invalid dimension hierarchy: ' | pHier;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;

If( Trim( pHier ) @= '' );
    ## use same name as Dimension. Since wildcards are allowed, this is managed inside the code below
EndIf;

## Default delimiter
If( pDelim     @= '' );
    pDelim     = '&';
EndIf;

### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### Create target dimension ###

If(DimensionExists( pTgtDim ) = 0 );
    DimensionCreate( pTgtDim );
Else;
    IF(pUnwind = 1 );
       nRet = ExecuteProcess('}bedrock.hier.unwind',
        'pLogOutput', pLogOutput,
        'pStrictErrorHandling', pStrictErrorHandling,
        'pDim', pTgtDim,
        'pHier', pTgtDim,
        'pConsol', '*',
        'pRecursive', 1
        );
    ELSEIF(pUnwind = 2 );
      # Do nothing;
    ELSE;
      DimensionDeleteAllElements( pTgtDim );
    EndIf;
EndIf;

### Set the target Sort Order ###
sSortElementsType     = CELLGETS( '}DimensionProperties', pSrcDim, 'SORTELEMENTSTYPE');
sSortElementsSense    = CELLGETS( '}DimensionProperties', pSrcDim, 'SORTELEMENTSSENSE');
sSortComponentsType   = CELLGETS( '}DimensionProperties', pSrcDim, 'SORTCOMPONENTSTYPE');
sSortComponentsSense  = CELLGETS( '}DimensionProperties', pSrcDim, 'SORTCOMPONENTSSENSE');

DimensionSortOrder( pTgtDim, sSortComponentsType, sSortComponentsSense, sSortElementsType , sSortElementsSense);


nSourceDimSize = DIMSIZ( pSrcDim );
nIndex = 1;
WHILE( nIndex <= nSourceDimSize );
  sElName = DIMNM( pSrcDim, nIndex);
  sElType = DTYPE( pSrcDim, sElName);
  
    DimensionElementInsert( pTgtDim, '', sElName, sElType );

  nIndex = nIndex + 1;
END;

### Assign Data Source ###
DatasourceNameForServer   = pSrcDim;
DatasourceNameForClient   = pSrcDim;
DataSourceType            = 'SUBSET';
DatasourceDimensionSubset = 'ALL';


### Replicate Attributes ###
# Note: DType on Attr dim returns "AS", "AN" or "AA" need to strip off leading "A"
sAttrDim        = '}ElementAttributes_' | pSrcDim;
sAttrLoc        = '}LocalizedElementAttributes_' | pSrcDim;
sAttrTargetDim  = '}ElementAttributes_' | pTgtDim;
sAttrLocTarget  = '}LocalizedElementAttributes_' | pTgtDim;

If( pAttr = 1 & DimensionExists( sAttrDim ) = 1 );
  nNumAttrs = DimSiz( sAttrDim );
  nCount = 1;
  While( nCount <= nNumAttrs );
    sAttrName = DimNm( sAttrDim, nCount );
    sAttCheck = SubSt( DTYPE( sAttrDim, sAttrName ), 1, 1 );
    IF (sAttCheck @= 'A');
    sAttrType = SubSt( DTYPE( sAttrDim, sAttrName ), 2, 1 );
    Else;
    sAttrType = sAttcheck;
    EndIf; 
      
      If ( DimensionExists( sAttrTargetDim ) = 0);
         AttrInsert(pTgtDim,'',sAttrName,sAttrType );
       ElseIF(DimIx(sAttrTargetDim, sAttrName) = 0);
         AttrInsert(pTgtDim,'',sAttrName,sAttrType );
      Endif;
        
    nCount = nCount + 1;
  End;
EndIf;


### End Prolog ###
#endregion
#region Metadata

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0.0~~##
################################################################################################# 


### Check for errors in prolog ###

If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### Add Elements to target dimension ###

sElType = DType( pSrcDim, vEle );
IF( sElType @= 'C' & ElCompN( pSrcDim, vEle ) > 0 );
    nChildren = ElCompN( pSrcDim, vEle );
    nCount = 1;
    While( nCount <= nChildren );
      sChildElement = ElComp( pSrcDim, vEle, nCount );
      sChildWeight = ElWeight( pSrcDim, vEle, sChildElement );
      DimensionElementComponentAdd( pTgtDim, vEle, sChildElement, sChildWeight );
      nCount = nCount + 1;
    End;
EndIf;

### End MetaData ###
#endregion
#region Data

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0.0~~##
################################################################################################# 


### Check for errors in prolog ###

If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### Replicate Attributes ###
# Note: DTYPE on Attr dim returns "AS", "AN" or "AA" need to strip off leading "A"

If( pAttr = 1 & DimensionExists( sAttrDim ) = 1 );

    nAttr = 1;
    While( nAttr <= nNumAttrs );
        sAttrName = DimNm( sAttrDim, nAttr );
        sAttCheck = SubSt( DTYPE( sAttrDim, sAttrName ), 1, 1 );
        IF (sAttCheck @= 'A');
        sAttrType = SubSt( DTYPE( sAttrDim, sAttrName ), 2, 1 );
        Else;
        sAttrType = sAttcheck;
        sMessage = pSrcDim | ' dimension contains invalid attribute - ' | sAttrName ;
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
        If( pStrictErrorHandling = 1 );
          ItemReject(Expand( cMsgErrorContent ));
        EndIf;  
        EndIf;
        If( CellIsUpdateable( sAttrTargetDim, vEle, sAttrName ) = 1 );
            If( sAttrType @= 'S' % sAttrType @= 'A' );
                #sAttrVal = AttrS( pSrcDim, vEle, sAttrName );
                sAttrVal = CellgetS('}ElementAttributes_'| pSrcDim, vEle, sAttrName);
                If( sAttrVal @<> '' );
                    If( sAttrType @= 'A' );
                        AttrPutS( sAttrVal, pTgtDim, vEle, sAttrName, 1 );
                    Else;
                        AttrPutS( sAttrVal, pTgtDim, vEle, sAttrName );
                    EndIf;
                EndIf;
            Else;
                #nAttrVal = AttrN( pSrcDim, vEle, sAttrName );
                nAttrVal = CellgetN('}ElementAttributes_'| pSrcDim, vEle, sAttrName);
                If( nAttrVal <> 0 );
                    AttrPutN( nAttrVal, pTgtDim, vEle, sAttrName );
                EndIf;
            EndIf;
        EndIf;
        # check for localized attributes
        If( CubeExists( sAttrLoc ) = 1 );
            nLang = 1;
            While( nLang <= nNumLang );
                sLang       = DimNm( cLangDim, nLang );
                If( sAttrType @= 'A' % sAttrType @= 'S' );
                    sAttrVal    = AttrS( pSrcDim, vEle, sAttrName );
                    sAttrValLoc = AttrSL( pSrcDim, vEle, sAttrName, sLang );
                    If( sAttrValLoc @= sAttrVal ); sAttrValLoc = ''; EndIf;
                Else;
                    nAttrVal    = AttrN( pSrcDim, vEle, sAttrName );
                    nAttrValLoc = AttrNL( pSrcDim, vEle, sAttrName, sLang );
                EndIf;
                If( CubeExists( sAttrLocTarget ) = 0 );
                    If( sAttrType @= 'A' );
                        AttrPutS( sAttrValLoc, pTgtDim, vEle, sAttrName, sLang, 1 );
                    ElseIf( sAttrType @= 'N' );
                        If( nAttrValLoc <> nAttrVal );
                            AttrPutN( nAttrValLoc, pTgtDim, vEle, sAttrName, sLang );
                        EndIf;
                    Else;
                        AttrPutS( sAttrValLoc, pTgtDim, vEle, sAttrName, sLang );
                    EndIf;
                ElseIf( CubeExists( sAttrLocTarget ) = 1 );
                    If( CellIsUpdateable( sAttrLocTarget, vEle, sLang, sAttrName ) = 1 );
                        If( sAttrType @= 'A' );
                            AttrPutS( sAttrValLoc, pTgtDim, vEle, sAttrName, sLang, 1 );
                        ElseIf( sAttrType @= 'N' );
                            If( nAttrValLoc <> nAttrVal );
                                AttrPutN( nAttrValLoc, pTgtDim, vEle, sAttrName, sLang );
                            EndIf;
                        Else;
                            AttrPutS( sAttrValLoc, pTgtDim, vEle, sAttrName, sLang );
                        EndIf;
                    EndIf;
                EndIf;
                nLang   = nLang + 1;
            End;
        EndIf;
        nAttr = nAttr + 1;
    End;

EndIf;

### End Data ###
#endregion
#region Epilog

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0.0~~##
################################################################################################# 


### Set the target Sort Order ###
  CELLPUTS( sSortElementsType, '}DimensionProperties', pTgtDim, 'SORTELEMENTSTYPE');
  CELLPUTS( sSortElementsSense, '}DimensionProperties', pTgtDim, 'SORTELEMENTSSENSE');
  CELLPUTS( sSortComponentsType, '}DimensionProperties', pTgtDim, 'SORTCOMPONENTSTYPE');
  CELLPUTS( sSortComponentsSense, '}DimensionProperties', pTgtDim, 'SORTCOMPONENTSSENSE');

### Destroy Source Subset ###

  If( SubsetExists( pSrcDim, cSubset ) = 1 );
    SubsetDestroy( pSrcDim, cSubset );
  EndIf;

##Clone all the Hierarchies except default hierarchy & Leaves
If( pHier @= '*' );
    sDim = pSrcDim;
    sHierDim = '}Hierarchies_' | sDim;
    sTargetHierarchy = '';
    nMax = DimSiz( sHierDim );
    nCtr = 1;
    While( nCtr <= nMax );
        sEle = DimNm( sHierDim, nCtr );
        nElength = Long(sEle);
        nElestart  = 0;
        nElestart = SCAN(':', sEle) + 1;
        If(nElestart > 1);
          vSourceHierarchy = SUBST(sEle,nElestart,nElength);
         If ( vSourceHierarchy @<> 'Leaves');
             nRet = ExecuteProcess('}bedrock.hier.clone',
               'pLogOutput', pLogOutput,
               'pStrictErrorHandling', pStrictErrorHandling,
               'pSrcDim', sDim,
               'pSrcHier', vSourceHierarchy,
               'pTgtDim', pTgtDim,
               'pTgtHier', vSourceHierarchy,
               'pAttr', pAttr,
               'pUnwind',pUnwind
               );
         Endif;
         sTargetHierarchy = sTargetHierarchy |':'|vSourceHierarchy;
        Endif;
        nCtr = nCtr + 1;
    End;
### Just one hierarchy specified in parameter
ElseIf( Scan( '*', pHier )=0 &  Scan( '?', pHier )=0 & Scan( pDelim, pHier )=0 & Trim( pHier ) @<> '' );
    sDim = pSrcDim;
    sHierDim = '}Hierarchies_' | sDim;
    sCurrHier = pHier;
    sCurrHierName = Subst( sCurrHier, Scan(':', sCurrHier)+1, Long(sCurrHier) );
    # Validate hierarchy name in sHierDim
    If( Dimix( sHierDim , sDim |':'| sCurrHier ) = 0 );
        sMessage = Expand('The "%sCurrHier%" hierarchy does NOT exist in the "%sDim%" dimension.');
        LogOutput( 'INFO' , Expand( cMsgInfoContent ) );
    ElseIf( sCurrHierName @= 'Leaves' );
        sMessage = 'Invalid  Hierarchy: ' | sCurrHier | ' will be skipped....';
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    ElseIf( sCurrHierName @<> sDim );
      If( pLogOutput = 1 );
        sMessage = Expand( 'Hierarchy "%sCurrHierName%" in Dimension "%sDim%" being processed....' );
        LogOutput( 'INFO', Expand( cMsgInfoContent ) );
      EndIf;
      nRet = ExecuteProcess('}bedrock.hier.clone',
       'pLogOutput', pLogOutput,
       'pStrictErrorHandling', pStrictErrorHandling,
       'pSrcDim', sDim,
       'pSrcHier', sCurrHierName,
       'pTgtDim', pTgtDim,
       'pTgtHier', sCurrHierName,
       'pAttr', pAttr,
       'pUnwind',pUnwind
      );
    Endif;
### Hierachy is a delimited list with no wildcards
ElseIf( Scan( '*', pHier )=0 &  Scan( '?', pHier )=0 & Trim( pHier ) @<> '' );
  
      # Loop through hierarchies in pHier
    sDim = pSrcDim;
    sHierarchies              = pHier;
    nDelimiterIndexA    = 1;
    sHierDim            = '}Hierarchies_'| sDim ;
    sMdxHier = '';
    While( nDelimiterIndexA <> 0 );
  
        nDelimiterIndexA = Scan( pDelim, sHierarchies );
        If( nDelimiterIndexA = 0 );
            sHierarchy   = sHierarchies;
        Else;
            sHierarchy   = Trim( SubSt( sHierarchies, 1, nDelimiterIndexA - 1 ) );
            sHierarchies  = Trim( Subst( sHierarchies, nDelimiterIndexA + Long(pDelim), Long( sHierarchies ) ) );
        EndIf;
        sCurrHier = sHierarchy;
        sCurrHierName = Subst( sCurrHier, Scan(':', sCurrHier)+1, Long(sCurrHier) );
        # Validate hierarchy name in sHierDim
        If( Dimix( sHierDim , sDim |':'| sCurrHier ) = 0 );
            sMessage = Expand('The "%sCurrHier%" hierarchy does NOT exist in the "%sDim%" dimension.');
            LogOutput( 'INFO' , Expand( cMsgInfoContent ) );
        ElseIf( sCurrHierName @= 'Leaves' );
            sMessage = 'Invalid  Hierarchy: ' | sCurrHier | ' will be skipped....';
            LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
        ElseIf( sCurrHierName @<> sDim );
          If( pLogOutput = 1 );
            sMessage = Expand( 'Hierarchy "%sCurrHierName%" in Dimension "%sDim%" being processed....' );
            LogOutput( 'INFO', Expand( cMsgInfoContent ) );
          EndIf;
          nRet = ExecuteProcess('}bedrock.hier.clone',
           'pLogOutput', pLogOutput,
           'pStrictErrorHandling', pStrictErrorHandling,
           'pSrcDim', sDim,
           'pSrcHier', sCurrHierName,
           'pTgtDim', pTgtDim,
           'pTgtHier', sCurrHierName,
           'pAttr', pAttr,
           'pUnwind',pUnwind
          );
        Endif;
    End;

### Hierachy has wildcards inside
ElseIf( Trim( pHier ) @<> '' );
  
      # Loop through hierarchies in pHier
    sDim = pSrcDim;
    sHierarchies              = pHier;
    nDelimiterIndexA    = 1;
    sHierDim            = '}Hierarchies_'| sDim ;
    sMdxHier = '';
    While( nDelimiterIndexA <> 0 );
  
        nDelimiterIndexA = Scan( pDelim, sHierarchies );
        If( nDelimiterIndexA = 0 );
            sHierarchy   = sHierarchies;
        Else;
            sHierarchy   = Trim( SubSt( sHierarchies, 1, nDelimiterIndexA - 1 ) );
            sHierarchies  = Trim( Subst( sHierarchies, nDelimiterIndexA + Long(pDelim), Long( sHierarchies ) ) );
        EndIf;
  
        # Create subset of Hierarchies using Wildcard
        sHierExp = '"'| sDim | ':' | sHierarchy|'"';
        sMdxHierPart = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sHierDim| '])},'| sHierExp | ')}';
        IF( sMdxHier @= ''); 
          sMdxHier = sMdxHierPart; 
        ELSE;
          sMdxHier = sMdxHier | ' + ' | sMdxHierPart;
        ENDIF;
    End;
  
    If( SubsetExists( sHierDim, cSubset ) = 1 );
        # If a delimited list of attr names includes wildcards then we may have to re-use the subset multiple times
        SubsetMDXSet( sHierDim, cSubset, sMdxHier );
    Else;
        # temp subset, therefore no need to destroy in epilog
        SubsetCreatebyMDX( cSubset, sMdxHier, sHierDim, 1 );
    EndIf;
  
    # Loop through subset of hierarchies created based on wildcard
    nCountHier = SubsetGetSize( sHierDim, cSubset  );
    While( nCountHier >= 1 );
        sCurrHier = SubsetGetElementName( sHierDim, cSubset , nCountHier );
        sCurrHierName = Subst( sCurrHier, Scan(':', sCurrHier)+1, Long(sCurrHier) );
        # Validate hierarchy name in sHierDim
        If( Dimix( sHierDim , sCurrHier ) = 0 );
            sMessage = Expand('The "%sCurrHier%" hierarchy does NOT exist in the "%sDim%" dimension.');
            LogOutput( 'INFO' , Expand( cMsgInfoContent ) );
        ElseIf( sCurrHierName @= 'Leaves' );
            sMessage = 'Invalid  Hierarchy: ' | sCurrHier | ' will be skipped....';
            LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
        ElseIf( sCurrHierName @<> sDim );
          If( pLogOutput = 1 );
            sMessage = Expand( 'Hierarchy "%sCurrHierName%" in Dimension "%sDim%" being processed....' );
            LogOutput( 'INFO', Expand( cMsgInfoContent ) );
          EndIf;
          nRet = ExecuteProcess('}bedrock.hier.clone',
           'pLogOutput', pLogOutput,
           'pStrictErrorHandling', pStrictErrorHandling,
           'pSrcDim', sDim,
           'pSrcHier', sCurrHierName,
           'pTgtDim', pTgtDim,
           'pTgtHier', sCurrHierName,
           'pAttr', pAttr,
           'pUnwind',pUnwind
          );
        Endif;
      
        nCountHier = nCountHier - 1;
    End;
Endif;

### Clone dimension subsets
If( pSub = 1);
  nCountSubs = DimSiz ('}Subsets_' | sDim);
  While ( nCountSubs >= 1 );
    sCurrSub = If( Scan( ':', DimNm ('}Subsets_' | sDim, nCountSubs)) = 0, DimNm ('}Subsets_' | sDim, nCountSubs), Subst( DimNm ('}Subsets_' | sDim, nCountSubs), Scan( ':', DimNm ('}Subsets_' | sDim, nCountSubs))+1, Long(DimNm ('}Subsets_' | sDim, nCountSubs))-Scan( ':', DimNm ('}Subsets_' | sDim, nCountSubs))));
    sCurrHier = If( Scan( ':', DimNm ('}Subsets_' | sDim, nCountSubs)) = 0, '', Subst(DimNm ('}Subsets_' | sDim, nCountSubs), 1, Scan( ':', DimNm ('}Subsets_' | sDim, nCountSubs))-1));

    ExecuteProcess('}bedrock.hier.sub.clone',
      'pLogOutput',0,
      'pStrictErrorHandling',0,
      'pSrcDim',sDim,
      'pSrcHier',sCurrHier,
      'pSrcSub',sCurrSub,
      'pTgtDim', pTgtDim,
      'pTgtHier', sCurrHier,
      'pTgtSub',sCurrSub,
      'pTemp',0,
      'pAlias','');
    nCountSubs = nCountSubs - 1;
  End;
Endif;
          

### Return code & final error message handling
If( nErrors > 0 );
    sMessage = 'the process incurred at least 1 error. Please see above lines in this file for more details.';
    nProcessReturnCode = 0;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    sProcessReturnCode = Expand( '%sProcessReturnCode% Process:%cThisProcName% completed with errors. Check tm1server.log for details.' );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    EndIf;
Else;
    sProcessAction = Expand( 'Process:%cThisProcName% has cloned the %pSrcDim% dimension into %pTgtDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion