#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.create.fromsubset', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pSrcDim', '', 'pSrcHier', '', 'pSubset', '',
    	'pTgtDim', '', 'pTgtHier', '',
    	'pAttr', 1, 'pUnwind', 0, 'pFlat', 0
	);
EndIf;
#EndRegion CallThisProcess

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4~~##
################################################################################################# 

#Region @DOC
# Description:
# This process will make a copy of an existing dimension subset, creating it as a new dimension hierarchy.

# Use case: Intended for Development but could be used in production too.
# 1. Create a new hierarchy for testing.
# 2. Create a new hierarchy to reflect new business needs.

# Note:
# Valid source dimension name (pSrcDim) and source subset (pSubset) are mandatory, otherwise the process will abort.
# If a source hierarchy name (pSrcHier) is specified, it needs to be valid, otherwise the process will abort.

# Caution:
# - Target hierarchy cannot be Leaves.
# - If the target Hierarchy already exists, then it will be overwritten.
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
cTempSub          = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pSrcDim:%pSrcDim%, pSrcHier:%pSrcHier%, pSubset:%pSubset%, pTgtDim:%pTgtDim%, pTgtHier:%pTgtHier%, pAttr:%pAttr%, pUnwind:%pUnwind%, pFlat:%pFlat%.';
cHierAttr         = 'Bedrock.Descendant';
cAttrVal          = 'Descendant';

## LogOutput parameters
IF ( pLogoutput = 1 );
  LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;
If( Scan( ':', pSrcDim ) > 0 & pSrcHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pSrcHier       = SubSt( pSrcDim, Scan( ':', pSrcDim ) + 1, Long( pSrcDim ) );
    pSrcDim        = SubSt( pSrcDim, 1, Scan( ':', pSrcDim ) - 1 );
EndIf;

If( Scan( ':', pTgtDim ) > 0 & pTgtHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pTgtHier       = SubSt( pTgtDim, Scan( ':', pTgtDim ) + 1, Long( pTgtDim ) );
    pTgtDim        = SubSt( pTgtDim, 1, Scan( ':', pTgtDim ) - 1 );
EndIf;

# Validate source dimension
IF( Trim( pSrcDim ) @= '' );
    nErrors = 1;
    sMessage = 'No source dimension specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

IF( DimensionExists( pSrcDim ) = 0 );
    nErrors = 1;
    sMessage = 'Invalid source dimension: ' | pSrcDim;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

### Check for errors before continuing
If( nErrors <> 0 );
   If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

# Validate source Hierarchy
IF(pSrcHier @= '');
    pSrcHier = pSrcDim;
ElseIf(HierarchyExists(pSrcDim, pSrcHier) = 0);
    nErrors = 1;
    sMessage = 'Invalid source hierarchy: ' | pSrcHier;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;

## Validate Source Subset
IF(HierarchySubsetExists( pSrcDim, pSrcHier, pSubset) = 0 );
    sMessage = 'No valid source subset: ' | pSubset;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ELSE;
    cSubset = pSubset;
ENDIF;

### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

## Validate target Dimension
If(pTgtDim @= '');
    pTgtDim = pSrcDim;
Endif;

IF( DimensionExists( pTgtDim ) = 0 );
   DimensionCreate(pTgtDim);
EndIf;

### Create target dimension Hierarchy ###
IF(pTgtHier @= '');
    pTgtHier = pTgtDim;
EndIf;


If( HierarchyExists(pTgtDim, pTgtHier) = 0 );
    HierarchyCreate( pTgtDim, pTgtHier );
Else;
    IF(pUnwind = 1 );
        ExecuteProcess( '}bedrock.hier.unwind', 'pLogOutput', 0,
            'pStrictErrorHandling', pStrictErrorHandling,
            'pDim', pTgtDim, 'pHier', pTgtHier, 'pConsol', '*',
            'pRecursive', 1
        );
    ELSEIF(
        pUnwind = 2 );
        #Do nothing
    ELSE;
        HierarchyDeleteAllElements( pTgtDim, pTgtHier );
    EndIf;
EndIf;

### Assign Data Source ###
DatasourceNameForServer = pSrcDim|':'|pSrcHier;
DatasourceNameForClient = pSrcDim|':'|pSrcHier;
DataSourceType = 'SUBSET';
DatasourceDimensionSubset = cSubset;

### Set Descendent attribute value
AttrDelete( pSrcDim, cHierAttr );
AttrInsert( pSrcDim, '', cHierAttr, 'S' );

# Disable excessive transaction logging of the attributes cube if it is logged
sAttrCube = '}ElementAttributes_' | pSrcDim;
nAttrCubeLogChanges = CubeGetLogChanges(sAttrCube);
If( nAttrCubeLogChanges = 1 );
   CubeSetLogChanges( sAttrCube, 0 );
EndIf;

nIndex = 1;
nLimit = HierarchySubsetGetSize( pSrcDim, pSrcHier, pSubset );
WHILE( nIndex <= nLimit);
    sElName = SubsetGetElementName( pSrcDim|':'|pSrcHier, pSubset, nIndex );
    ElementAttrPuts( cAttrVal, pSrcDim, pSrcHier, sElName, cHierAttr );
    sElType = ElementType( pSrcDim, pSrcHier, sElName );
    HierarchyElementInsert(pTgtDim, pTgtHier, '',sElName, sELType);
    nIndex = nIndex + 1;
END;

# Re-enable transaction logging setting of the attributes cube if required
If( nAttrCubeLogChanges = 1 );
   CubeSetLogChanges( sAttrCube, 1 );
EndIf;

### Replicate Attributes ###
# Note: DType on Attr dim returns "AS", "AN" or "AA" need to strip off leading "A"
 
sAttrDim = '}ElementAttributes_' | pSrcDim;
sLastAttr = '';
If( pAttr = 1 & DimensionExists( sAttrDim ) = 1 );
    nNumAttrs = DimSiz( sAttrDim );
    nCount = 1;
    While( nCount <= nNumAttrs );
        sAttrName = DimNm( sAttrDim, nCount );
        sAttrType = SubSt(DType( sAttrDim, sAttrName ), 2, 1 );
        AttrInsert( pTgtDim, sLastAttr, sAttrName, sAttrType );
        sLastAttr = sAttrName;
        nCount = nCount + 1;
    End;
EndIf;
 
### End Prolog ###
#endregion
#region Metadata

#****Begin: Generated Statements***
#****End: Generated Statements****

### Check for errors in prolog ###
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

If (pFlat = 1);
    ##Creating the Flat hierarchy subset in the target dimension
    sElType = ElementType(pSrcDim, pSrcHier, vElement);
    ## Add the element to the target dimension.
    ## 'C' elements can't be inserted as 'N' elements in the same dimension
    IF(pTgtdim @= pSrcDim);
        IF(sElType @<> 'C' );
            HierarchyElementInsert( pTgtDim, pTgtHier, '', vElement, sElType );
        Else;
            If( pLogOutput = 1 );
                sMessage = 'Name conflict! Cannot create leaf element %vElement% in dimension %pTgtDim% as C element with same name already exists.';
                LogOutput( 'WARN', Expand( cMsgErrorContent ) );
            EndIf;
        EndIf;
    Else;
        IF(sElType @= 'C' );
            HierarchyElementInsert( pTgtDim, pTgtHier, '', vElement, 'N' );
        Else;
            HierarchyElementInsert( pTgtDim, pTgtHier, '', vElement, sElType );
        EndIf;
    EndIf;
Else;
    nIndex = 1;
    nLimit = ElementComponentCount( pSrcDim, pSrcHier, vElement );
    WHILE( nIndex <= nLimit );
        sElName = ElementComponent( pSrcDim, pSrcHier, vElement, nIndex );
        sDecendant = ElementAttrS(pSrcDim, pSrcHier, sElName, cHierAttr);
        IF(
            sDecendant @= cAttrVal);
            nElWeight = ElementWeight( pSrcDim, pSrcHier, vElement, sElName );
            HierarchyElementComponentAdd( pTgtDim, pTgtHier, vElement, sElName, nElWeight );
        ENDIF;
        nIndex = nIndex + 1;
    END;
Endif;

### End MetaData ###
#endregion
#region Data

#****Begin: Generated Statements***
#****End: Generated Statements****


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

If( pAttr = 1 );

    nCount = 1;
    While( nCount <= nNumAttrs );
        sAttrName = DimNm( sAttrDim, nCount );
        sAttrType = SubSt( DTYPE( sAttrDim, sAttrName ), 2, 1 );
        If( sAttrType @= 'S' % sAttrType @= 'A' );
            sAttrVal = ElementAttrS(pSrcDim, pSrcHier, vElement, sAttrName);
            If( sAttrVal @<> '' );
                If( pStrictErrorHandling = 0 & CellIsUpdateable(sAttrDim, pTgtHier:vElement, sAttrName) = 0 );
                    #skip
                ElseIf( sAttrType @= 'A' );
                    ElementAttrPutS( sAttrVal, pTgtDim, pTgtHier, vElement, sAttrName, 1 );
                Else;
                    ElementAttrPutS( sAttrVal, pTgtDim, pTgtHier, vElement, sAttrName );
                EndIf;
            EndIf;
        Else;
            nAttrVal = ElementAttrN(pSrcDim, pSrcHier, vElement, sAttrName);
            If( nAttrVal <> 0 );
                If( pStrictErrorHandling = 0 & CellIsUpdateable(sAttrDim, pTgtHier:vElement, sAttrName) = 0 );
                    #skip
                Else;
                    ElementAttrPutN( nAttrVal, pTgtDim, pTgtHier, vElement, sAttrName );
                EndIf;
            EndIf;
        EndIf;
        nCount = nCount + 1;
    End;

  EndIf;

### End Data ###
#endregion
#region Epilog

#****Begin: Generated Statements***
#****End: Generated Statements****

### Set Target dimension sort order ###

If(pSrcDim @= pSrcHier);
    sSourceElement = pSrcDim;
Else;
    sSourceElement = pSrcDim|':'|pSrcHier;
Endif;
If(pTgtDim @= pTgtHier);
    sTargetElement = pTgtDim;
Else;
    sTargetElement = pTgtDim|':'|pTgtHier;
Endif;

sCube = '}DimensionProperties';
IF(CubeExists ( sCube ) = 1 );
  sEleMapping = '}Dimensions' |'¦'|sSourceElement|'->'|sTargetElement;
  ExecuteProcess( '}bedrock.cube.data.copy',
  'pLogOutput', pLogOutput,
  'pStrictErrorHandling', pStrictErrorHandling,
  'pCube', sCube,
  'pSrcView', '',
  'pTgtView', '',
  'pFilter',  '',
  'pEleMapping', sEleMapping,
  'pMappingDelim','->',
  'pFactor', 1,
  'pDimDelim', '&',
  'pEleStartDelim', '¦',
  'pEleDelim', '+',
  'pSuppressRules', 0 ,
  'pCumulate', 0 ,
  'pZeroSource', 0, 
  'pZeroTarget', 1,
  'pTemp', 1
   );
ENDIF;
  
sCube = '}HierarchyProperties';
IF(CubeExists ( sCube ) = 1 );
  sEleMapping = '}Dimensions' |'¦'|sSourceElement|'->'|sTargetElement;
  ExecuteProcess( '}bedrock.cube.data.copy',
  'pLogOutput', pLogOutput,
  'pStrictErrorHandling', pStrictErrorHandling,
  'pCube', sCube,
  'pSrcView', '',
  'pTgtView', '',
  'pFilter',  '',
  'pEleMapping', sEleMapping,
  'pMappingDelim','->',
  'pFactor', 1,
  'pDimDelim', '&',
  'pEleStartDelim', '¦',
  'pEleDelim', '+',
  'pSuppressRules', 0 ,
  'pCumulate', 0 ,
  'pZeroSource', 0, 
  'pZeroTarget', 1,
  'pTemp', 1
   );
ENDIF;


### Set Descendent attribute value
AttrDelete( pSrcDim, cHierAttr );
If( pAttr = 1 );
    AttrDelete( pTgtDim, cHierAttr );
ENDIF;

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully cloned dimension:hierarchy %pSrcDim%:%pSrcHier% to %pTgtDim%:%pTgtHier% based on the %pSubset% subset.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion