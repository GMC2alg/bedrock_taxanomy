#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.consol.flat.create', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '',
    	'pSrcConsol', '', 'pTgtConsol', '', 'pWeight', 1
	);
EndIf;
#EndRegion CallThisProcess

#****Begin: Generated Statements***
#****End: Generated Statements****

### Start Prolog ###

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

#Region @DOC
# Description:
# This process will add consolidated element pTgtConsol in the dimension hierarchy. 

# Use case: Intended for Development but could be used in production too.
# 1/ Create a new hierarchy for testing.
# 2/ Create a new hierarchy to reflect new business needs.

# Note:
# If source parent element (pSrcConsol) is defined then all its leaf elements will be copied to target
# parent element consolidation. Otherwise all leaf elements in the dimension hierarchy will be 
# added into target parent element consolidation.
# Valid source dimension name (pDim), hierarchy (pHier) and taget parent element (pTgtConsol)
# are mandatory otherwise the process will abort.
# Weight of all added elements into target parent consolidation will be assigned accoring to
# pWeight parameter value.

# Caution: If the target element pTgtConsol exists in the hierarchy, then it will be overwritten.
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
cMsgErrorContent= 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo        = 'Process:%cThisProcName% run with parameters pDim=%pDim%, pHier=%pHier%, pSrcConsol=%pSrcConsol%, pTgtConsol=%pTgtConsol%.';

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

If( Scan( ':', pDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pDim, Scan( ':', pDim ) + 1, Long( pDim ) );
    pDim        = SubSt( pDim, 1, Scan( ':', pDim ) - 1 );
EndIf;

IF( Trim( pDim ) @= '' );
    ## No dimension nominated.
    nErrors = 1;
    sMessage = 'No dimension specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( DimensionExists( pDim) = 0 );
    ## Dimension does not exist in the model.
    nErrors = 1;
    sMessage = 'The dimension does not exist in the model: ' | pDim;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

IF( SUBST( pDim, 1, 1 ) @= '}' );
    ## Nominated dimension is a system dimension.
    nErrors = 1;
    sMessage = 'Can not modify a system dimension with this Bedrock.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate hierarchy
IF( Trim( pHier  ) @= '' );
    pHier = pDim;
EndIf;

IF( HierarchyExists(pDim, pHier ) = 0 % pHier @= 'Leaves');
    nErrors = 1;
    sMessage = 'Invalid dimension Hierarchy: ' | pDim |':'|pHier;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate consolidation Parameter
pTgtConsol = Trim( pTgtConsol );
If( pTgtConsol @= '');
    nErrors = 1;
    sMessage = 'No target parent element supplied.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( pTgtConsol @= pSrcConsol );
    nErrors = 1;
    sMessage = 'Target and source consolidations can''t be the same elements: ' | pTgtConsol |':'| pSrcConsol;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;

## Validate Member Consolidation
pSrcConsol = Trim( pSrcConsol );
IF( pSrcConsol @<> '' );
    ## A source consolidation has been nominated.
    IF(  ElementIndex( pDim, pHier, pSrcConsol ) = 0 );
    ## The Member Consolidation does not exist in the dimension.
        nErrors = 1;
        sMessage = Expand('Source consolidation %pSrcConsol% does not exist in the dimension:hierarchy: %pDim%:%pHier%.');
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    ENDIF;
EndIf;

### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### UNWIND CONSOLIDATION ###
IF( ElementIndex( pDim, pHier, pTgtConsol ) > 0 );
  ExecuteProcess('}bedrock.hier.unwind'
    ,'pStrictErrorHandling', pStrictErrorHandling
    ,'pDim', pDim
    ,'pHier',pHier
    ,'pConsol', pTgtConsol
    ,'pRecursive', 0
  );
Else;
    HierarchyElementInsert( pDim, pHier, '', pTgtConsol, 'C' );
ENDIF;

### Assign Datasource ###
DatasourceNameForServer = pDim|':'|pHier;
DataSourceDimensionSubset = 'All';

### End Prolog ###
#endregion
#region Metadata

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0.0~~##
################################################################################################# 

### Start Metadata ###

### VALIDATE RECORD ###

sElType = ElementType( pDim, pHier, vElement );
IF( sElType @<> 'N' );
  ## The element is not a 'N' element.
  ITEMSKIP;
ENDIF;

IF( pSrcConsol @<> '' );
  ## A member consolidation is been used.
  IF( ElementIsAncestor( pDim, pHier, pSrcConsol, vElement ) = 0 );
  ## The element is not a member of the nominated consolidation.
    ITEMSKIP;
  ENDIF;
ENDIF;  

### BUILD CONSOLIDATION ###

If( nErrors = 0 );
  HierarchyElementComponentAdd( pDim, pHier, pTgtConsol, vElement, pWeight );
EndIf;

### End Metadata ###
#endregion
#region Data

#****Begin: Generated Statements***
#****End: Generated Statements****
#endregion
#region Epilog

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0.0~~##
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully created a consolidation %pTgtConsol% in the hierarchy %pDim%:%pHier%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;
#endregion