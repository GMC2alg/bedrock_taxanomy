#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.create.byelement', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '', 'pSub', '',
    	'pEle', '', 'pDelim', '&',
    	'pAddToSubset', 0, 'pExpandConsol', 1,
    	'pAlias', '', 'pTemp', 1
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
# This process will create a static subset in a Hierarchy of target Dimension based on a list of
# supplied elements.

# Note:
# - pAddToSubset: If the specified subset already exists then this parameter will control whether elements will
#                 be added to the existing subset (value 1) or a new subset will be created (value 0).
# - pExpandConsol: If the specified list of elements contains consolidated elements they will be replaced with 
#                 their leaf level descendants
# Caution: Process doesn't accept wildcards in element names.
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName       = GetProcessName();
cUserName           = TM1User();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cTempSub            = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pSub:%pSub%, pEle:%pEle%, pDelim:%pDelim%, pAddToSubset:%pAddToSubset%, pExpandConsol:%pExpandConsol%, pAlias:%pAlias%, pTemp:%pTemp%.'; 
cAttributeDim       = '}ElementAttributes_' | pDim;

## LogOutput parameters
IF ( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;


### Validate Parameters ###

nErrors = 0;

If( Scan( ':', pDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pDim, Scan( ':', pDim ) + 1, Long( pDim ) );
    pDim        = SubSt( pDim, 1, Scan( ':', pDim ) - 1 );
EndIf;

# Validate dimension

If( Trim( pDim ) @= '' );
  nErrors = 1;
  sMessage = 'No dimension specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
If( DimensionExists( pDim ) = 0 );
  nErrors = 1;
  sMessage = 'Invalid dimension: ' | pDim;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate Hierarchy

IF(pHier @= 'Leaves' );
  nErrors = 1;
  sMessage = 'Invalid  Hierarchy: ' | pDim |':'|pHier;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

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
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate elements
If( Trim( pEle ) @= '' );
  nErrors = 1;
  sMessage = 'No elements specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate delimiter
If( pDelim @= '' );
  pDelim = '&';
EndIf;

# Validate add to subset
If( pAddToSubset <> 0 & pAddToSubset <> 1 );
  nErrors = 1;
  sMessage = 'Invalid value for pAddToSubset: ' | NumberToString( pAddToSubset ) | '. Valid values are 0 and 1';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate expand consolidations
If( pExpandConsol <> 0 & pExpandConsol <> 1 & pExpandConsol <> 2 );
  nErrors = 1;
  sMessage = 'Invalid value for pExpandConsol: ' | NumberToString( pExpandConsol ) | '. Valid values are 0, 1 and 2';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate Alias
sDimAttr = '}ElementAttributes_' | pDim;
IF(pAlias @<> '' );
  IF(DimensionExists( sDimAttr ) = 0 );
    sMessage = 'No attributes exist for the dimension: ' | pDim;
    pAlias = '';
  EndIf;

  IF(DIMIX( sDimAttr, pAlias ) = 0 );
    sMessage = 'The Alias: ' | pAlias | ' does not exist in the dimension: ' | pDim;
    pAlias = '';
  EndIf;

  IF(
  DTYPE( sDimAttr, pAlias ) @<> 'AA' );
    sMessage = 'The Alias: ' | pAlias | ' is not an Alias in the dimension: ' | sDimAttr;
    pAlias = '';
  EndIf;
ENDIF;

## Validate pTemp
IF( pTemp <> 0 & pTemp <> 1 );
    nErrors = 1;
    sMessage = 'Wrong parameter pTemp value (only 0 or 1 accepted).';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

### Prepare subset ###
If( HierarchySubsetExists( pDim, pHier, pSub ) = 1 );
  If( pAddtoSubset <> 1 );
    HierarchySubsetDeleteAllElements( pDim, pHier, pSub );
  EndIf;
Else;
  If( nErrors = 0 );
    HierarchySubsetCreate( pDim, pHier, pSub, pTemp );
  EndIf;
EndIf;

### Assign Alias to subset
IF(pAlias @<> '' );
  HierarchySubsetAliasSet( pDim, pHier, pSub, pAlias );
ENDIF;

### Insert elements ###

sElements = pEle;
nDelimIndex = 1;

# Split filter into separate dimensions
While( nDelimIndex <> 0 & Long( sElements ) > 0 );

  nDelimIndex = Scan( pDelim, sElements );
  If( nDelimIndex <> 0 );
    sElement = Trim( SubSt( sElements, 1, nDelimIndex - 1 ) );
    sElements = Trim( SubSt( sElements, nDelimIndex + Long( pDelim ), Long( sElements ) ) );
  Else;
    sElement = Trim( sElements );
  EndIf;

  If( ElementIndex( pDim, pHier, sElement ) <> 0 );
    If( nErrors = 0 );
      IF( ElementLevel( pDim, pHier, sElement) > 0 );
        If( pExpandConsol = 1 );
          ExecuteProcess('}bedrock.hier.sub.create',
                         'pStrictErrorHandling', pStrictErrorHandling,
                         'pDim', pDim,
                         'pHier',pHier,
                         'pSub', pSub,
                         'pConsol', sElement,
                         'pExclusions', '',
                         'pDelim', pDelim,
                         'pAddToSubset', 1,
                         'pAlias', '',
                         'pTemp', pTemp
                        );
        ElseIf( pExpandConsol = 2 );
          ExecuteProcess('}bedrock.hier.sub.create',
                         'pStrictErrorHandling', pStrictErrorHandling,
                         'pDim', pDim,
                         'pHier',pHier,
                         'pSub', pSub,
                         'pConsol', sElement,
                         'pLevelFrom', 0,
                         'pLevelTo', 0,
                         'pExclusions', '',
                         'pDelim', pDelim,
                         'pAddToSubset', 1,
                         'pAlias', '',
                         'pTemp', pTemp
                        );
        Else;
          HierarchySubsetElementInsert( pDim, pHier, pSub, sElement, 0 );
        EndIf;
      ELSE;
        HierarchySubsetElementInsert( pDim, pHier, pSub, sElement, 0 );
      ENDIF;
    EndIf;
  EndIf;
End;


### End Prolog ###
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully created subset %pSub% from dimension %pDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion