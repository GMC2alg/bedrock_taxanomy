#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.create.bymdx', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '', 'pSub', '',
    	'pMDXExpr', '',
    	'pConvertToStatic', 1, 'pTemp', 1
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
# This process will Create a dynamic subset from an MDX expression that evaluates to a non-empty set in the specified dimension.

# Use case: Intended for Production & Development
#1/ Create a dynamic subset for use in a view

# Note:
# Naturally, valid dimension name (pDim) are mandatory otherwise the process will abort.
# If the MDX does not compile or produces an empty set, the process will error.
# If convert to static (pConvertToStatic) is set to 1 then the MDX subset will be replaced by a static subset.
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName   = GetProcessName();
cTimeStamp      = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt      = NumberToString( INT( RAND( ) * 1000 ));
cTempSub        = cThisProcName | '_' | cTimeStamp | '_' | cRandomInt;
cTempFile       = GetProcessErrorFileDirectory | cTempSub | '.csv';
cUserName       = TM1User();
cMsgErrorLevel  = 'ERROR';
cMsgErrorContent= 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo        = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pSub:%pSub%, pMDXExpr:%pMDXExpr%, pConvertToStatic:%pConvertToStatic%, pTemp:%pTemp%, pAlias:%pAlias%.' ;
sMDXExpr        = pMDXExpr;

## LogOutput parameters
IF( pLogoutput  = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###

nErrors         = 0;

If( Scan( ':', pDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pDim, Scan( ':', pDim ) + 1, Long( pDim ) );
    pDim        = SubSt( pDim, 1, Scan( ':', pDim ) - 1 );
EndIf;

# Validate dimension
If( Trim( pDim )  @= '' );
    nErrors     = 1;
    sMessage    = 'No dimension specified';
    DataSourceType= 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( Scan( ':', pDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pDim, Scan( ':', pDim ) + 1, Long( pDim ) );
    pDim        = SubSt( pDim, 1, Scan( ':', pDim ) - 1 );
EndIf;

If( DimensionExists( pDim ) = 0 );
    nErrors     = 1;
    sMessage    = 'Invalid dimension: ' | pDim;
    DataSourceType = 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate hierarchy
If( Trim( pHier ) @= '' );
    sHier = pDim;
Else;
    sHier = pHier;
EndIf;

IF(HierarchyExists(pDim, pHier ) = 0 );
    nErrors = 1;
    sMessage = 'Invalid dimension Hierarchy: ' | pDim |':'|pHier;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate subset
If( Trim( pSub ) @= '' );
    nErrors = 1;
    sMessage = 'No subset specified';
    DataSourceType = 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate MDX
If( Trim( sMDXExpr ) @= '' );
    nErrors = 1;
    sMessage = 'No MDX expression specified.';
    DataSourceType = 'NULL';
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
    DimIx ( Expand ( '}ElementAttributes_%pDim%' ), pAlias ) = 0
);
  nErrors = 1;
  sMessage = 'Alias does not exist in dimension %pDim%.';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;  

# Validate alias attribute name is actually an alias
If ( pAlias @<> '' & 
    Dtype ( Expand ( '}ElementAttributes_%pDim%' ), pAlias ) @<> 'AA'  
);
  nErrors = 1;
  sMessage = 'Attribute %pAlias% is not an alias in dimension %pDim%.';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;


### Create Subset ###
If( nErrors = 0 );
  If( ElementCount( pDim, sHier ) = 0 & pConvertToStatic <> 0);
    HierarchySubsetCreate( pDim, sHier, pSub );
  Else;
    If( HierarchySubsetExists( pDim,sHier, pSub ) = 1 );
        HierarchySubsetMDXSet( pDim, sHier, pSub, sMDXExpr );
    Else;
        SubsetCreateByMDX( pSub, sMDXExpr, pDim|':'|sHier, pTemp );
    EndIf;
    If( pConvertToStatic = 1 );
        HierarchySubsetElementInsert( pDim, sHier, pSub, ElementName( pDim, sHier, 1 ), 1 );
        HierarchySubsetElementDelete( pDim, sHier, pSub, 1 );
    EndIf;
  EndIf;
  
  # Set Alias
  If ( pAlias @<> '' );
      If ( pDim @= sHier );
          SubsetAliasSet( pDim, pSub, pAlias);
      Else;
          SubsetAliasSet( pDim | ':' | sHier, pSub, pAlias);
      EndIf;
  EndIf;
EndIf;


### End Prolog ###
#endregion
#region Metadata

#****Begin: Generated Statements***
#****End: Generated Statements****

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

### Destroy Temporary Subset ###

If( pConvertToStatic = 1 & pTemp = 0 );

  If( HierarchySubsetExists( pDim , pHier, cTempSub) = 1 );
    HierarchySubsetDestroy( pDim, pHier, cTempSub );
  EndIf;

EndIf;

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully created subset %pSub% from dimension %pDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion