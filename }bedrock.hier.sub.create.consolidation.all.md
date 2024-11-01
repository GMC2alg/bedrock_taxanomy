#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.create.consolidation.all', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '', 'pSub', '', 'pConsol', '',
    	'pAddToSubset', 0, 'pExclusions', '',
    	'pDelim', '&', 'pAlias', '', 'pTemp', 1
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
# This process will create a static subset in a Hierarchy of target Dimension that consists of
# all descendants of a specified consolidated element, including consolidated elements.

# Note:
# - pExclusions: If pExclusions is specified then the elements (separated by a delimiter) will be
#   excluded from the subset. Wildcards characters `*` and `?` are accepted.
# - pAddToSubset : If the specified subset already exists then this parameter will control whether
#   elements will be added to the existing subset (value 1) or a new subset will be created
#   (value 0).

# Caution: Number of consolidated levels that are processed is limited to maximum of 99.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###

cThisProcName = GetProcessName();
cUserName         = TM1User();
cMsgErrorLevel    = 'ERROR';
cMsgErrorShell = Expand('Executing process %cThisProcName% has failed.');
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pSub:%pSub%, pConsol:%pConsol%, pAddToSubset:%pAddToSubset%, pExclusions:%pExclusions%, pDelim:%pDelim%, pAlias:%pAlias%, pTemp:%pTemp%.' ;

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

nErrors = 0;

## Validate pTemp
IF( pTemp <> 0 & pTemp <> 1 );
    nErrors = 1;
    sMessage = 'Wrong parameter pTemp value (only 0 or 1 accepted).';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

nRet = 
ExecuteProcess( '}bedrock.hier.sub.create', 'pLogOutput', pLogOutput,
    'pStrictErrorHandling', pStrictErrorHandling,
    'pDim', pDim, 'pHier', pHier, 'pSub', pSub, 'pConsol', pConsol,
    'pAttr', '', 'pAttrValue', '',
    'pLevelFrom', 0, 'pLevelTo', 999,
    'pExclusions', pExclusions, 'pDelim', pDelim,
    'pAddToSubset', pAddToSubset,
    'pAlias', pAlias, 'pTemp', pTemp
);
                            
IF ( nRet <> ProcessExitNormal() );
  sMessage = cMsgErrorShell;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
  ProcessError();
ENDIF;

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