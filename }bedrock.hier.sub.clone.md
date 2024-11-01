#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.clone', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pSrcDim', '', 'pSrcHier', '', 'pSrcSub', '',
    	'pTgtDim', '', 'pTgtHier', '', 'pTgtSub', '',
    	'pTemp', 1
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
# This process will copy a subset from a Hierarchy in source Dimension to a Hierarchy in target
# Dimension.

# Note:
# Valid source dimension name (pSrcDim), source (pSrcSub) and target subset (pTgtSub) are 
# mandatory otherwise the process will abort.

# Caution:
# - Target hierarchy cannot be `Leaves`.
# - If the target dimension Hierarchy exists then it will be overwritten.
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName   = GetProcessName();
cUserName       = TM1User();
cTimeStamp      = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt      = NumberToString( INT( RAND( ) * 1000 ));
cTempSub        = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel  = 'ERROR';
cMsgErrorContent= 'Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo        = 'Process:%cThisProcName% run with parameters pSrcDim:%pSrcDim%, pSrcHier:%pSrcHier%, pSrcSub:%pSrcSub%, pTgtDim:%pTgtDim%, pTgtHier:%pTgtHier%, pTgtSub:%pTgtSub%, pTemp:%pTemp%, pAlias:%pAlias%.' ; 

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

If( Scan( ':', pSrcDim ) > 0 & pSrcHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pSrcHier       = SubSt( pSrcDim, Scan( ':', pSrcDim ) + 1, Long( pSrcDim ) );
    pSrcDim        = SubSt( pSrcDim, 1, Scan( ':', pSrcDim ) - 1 );
EndIf;

# Validate Source dimension
If( Trim( pSrcDim ) @= '' );
    nErrors = 1;
    sMessage = 'No source dimension specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( DimensionExists( pSrcDim ) = 0 );
    nErrors = 1;
    sMessage = 'Dimension ' | pSrcDim | ' does not exist.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate Source Hierarchy
If( Trim( pSrcHier ) @= '' );
    pSrcHier = Trim( pSrcDim );
EndIf;

If( HierarchyExists( pSrcDim , pSrcHier ) = 0 );
    nErrors = 1;
    sMessage = 'The Hierachy ' | pSrcHier | ' does not exist.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate Source subset
If( Trim( pSrcsub ) @= '' );
    nErrors = 1;
    sMessage = 'No source subset specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( HierarchySubsetExists( pSrcDim , pSrcHier, pSrcsub ) = 0 );
    nErrors = 1;
    sMessage = 'Invalid source subset : ' | pSrcDim |':'| pSrcHier |':'| pSrcSub;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( Scan( ':', pTgtDim ) > 0 & pTgtHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pTgtHier       = SubSt( pTgtDim, Scan( ':', pTgtDim ) + 1, Long( pTgtDim ) );
    pTgtDim        = SubSt( pTgtDim, 1, Scan( ':', pTgtDim ) - 1 );
EndIf;

# Validate target dimension
If( Trim( pTgtDim ) @= '' );
    pTgtDim = Trim( pSrcDim );
ElseIf( DimensionExists( pTgtDim ) = 0 );
    nErrors = 1;
    sMessage = 'Invalid target dimension: ' | pTgtDim;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;


# Validate Target Hierarchy
If( Trim( pTgtHier ) @= '' );
    pTgtHier = pTgtDim;
ElseIf( HierarchyExists( pTgtDim, pTgtHier ) = 0 );
    nErrors = 1;
    sMessage = 'The target Hierachy ' | pTgtHier | ' does not exist.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate subset
If( Trim( pTgtSub ) @= '' );
    nErrors = 1;
    sMessage = 'No target subset specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate pTemp
IF( pTemp <> 0 & pTemp <> 1 );
    nErrors = 1;
    sMessage = 'Wrong parameter pTemp value (only 0 or 1 accepted).';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate Alias exists
If ( pAlias @<> '' & 
    DimIx ( Expand ( '}ElementAttributes_%pTgtDim%' ), pAlias ) = 0
);
  nErrors = 1;
  sMessage = 'Alias does not exist in dimension %pTgtDim%.';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;  

# Validate alias attribute name is actually an alias
If ( pAlias @<> '' & 
    Dtype ( Expand ( '}ElementAttributes_%pTgtDim%' ), pAlias ) @<> 'AA'
);
  nErrors = 1;
  sMessage = 'Attribute %pAlias% is not an alias in dimension %pTgtDim%.';
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

### Create Target Subset ###
If( HierarchySubsetExists( pTgtDim, pTgtHier, pTgtsub ) = 1 );
    HierarchySubsetDeleteAllElements( pTgtDim, pTgtHier, pTgtsub );
Else;
    HierarchySubsetCreate( pTgtDim, pTgtHier, pTgtsub, pTemp );
EndIf;

### Set Alias ###
If ( pAlias @<> '' );
    If ( pTgtDim @= pTgtHier );
        SubsetAliasSet( pTgtDim, pTgtsub, pAlias);
    Else;
        SubsetAliasSet( pTgtDim | ':' | pTgtHier, pTgtsub, pAlias);
    EndIf;
EndIf;

# HierarchySubsetMDXGet not returning anything. Thought it might also return alias used in source subset
sMDX = HierarchySubsetMDXGet(pSrcDim, pSrcHier, pSrcSub);

nElementPosition = 0;

### Set data source for process ###
DatasourceType              = 'SUBSET';
DatasourceNameForServer     = pSrcDim | ':' | pSrcHier;
DatasourceDimensionSubset   = pSrcsub;
#endregion
#region Metadata

#****Begin: Generated Statements***
#****End: Generated Statements****


################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 


IF( pTgtDim @= pSrcDim & pTgtHier @= pSrcHier);
    nElementPosition = nElementPosition + 1;
ElseIF( ElementIndex( pTgtDim, pTgtHier,vEle ) > 0 );
    nElementPosition = nElementPosition + 1;
Else;
    ItemReject( Expand( 'Cannot insert into subset. Element  %vEle% does not exist in target dimension:Hierarchy %pTgtDim%:%pTgtHier%.' ) );
EndIF;

HierarchySubsetElementInsert( pTgtDim , pTgtHier, pTgtSub , vEle , nElementPosition );
#endregion
#region Data

#****Begin: Generated Statements***
#****End: Generated Statements****
#endregion
#region Epilog

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully cloned the %pTgtDim%:%pTgtHier%:%pTgtSub% subset from %pSrcDim%:%pSrcHier%:%pSrcSub%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion