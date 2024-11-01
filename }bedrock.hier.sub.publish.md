#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.publish', 'pLogOutput', pLogOutput, 'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '', 'pSub', '',
    	'pOverwrite', 0
	);
EndIf;
#EndRegion CallThisProcess

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0.0~~##
################################################################################################# 

#Region @DOC
# Description:
# This process converts a private subset to a public subset for the named client.
#
# Use case: Intended for development/prototyping or production.
# 1. Make private subset public to enable public consumption.
#
# Note:
# * A valid dimension name pDim is mandatory otherwise the process will abort.
# * Also, a valid subset name pSub _belonging to the user running the process_ is mandatory otherwise the process will abort.
# * This process must be run by the user owning the private subset; it canot be run by another user.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName       = GetProcessName();
cUserName           = TM1User();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cTempSubset         = cThisProcName | '_' | cTimeStamp | '_' | cRandomInt;
cTempFile           = GetProcessErrorFileDirectory | cTempSubset | '.csv';
sMessage            = 	'';
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pSub:%pSub%, pSubPublish:%pSubPublish%, pOverwrite:%pOverwrite%.' ;

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

# create friendly name for user handle
If( DimIx( '}ElementAttributes_}Clients', '}TM1_DefaultDisplayValue' ) > 0 );
    pClient = AttrS( '}Clients', cUserName, '}TM1_DefaultDisplayValue' );
    If( pClient @= '' );
        pClient = cUserName;
    EndIf;
Else;
    pClient = cUserName;
EndIf;

# Validate Dimension & Hierarchy
If( Scan(':', pDim ) > 0 );
    pHier       = SubSt( pDim, Scan(':', pDim )+1, Long(pDim) - Scan(':', pDim ) );
    pDim        = SubSt( pDim, 1, Scan(':', pDim )-1 );
EndIf;
If( pHier @= '' );
    pHier = pDim;
EndIf;
If( Trim( pDim ) @= '' );
    sMessage    = 'No dimension specified';
    nErrors     = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( DimensionExists( pDim ) = 0 );
    sMessage = Expand('Dimension %pDim% does not exist on server');
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( HierarchyExists( pDim, pHier ) = 0 );
    sMessage = Expand('Hierarchy %pHier% does not exist in dimension %pDim%');
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
If( pHier @= pDim );
    sDimHier = pDim;
Else;
    sDimHier = Expand('%pDim%:%pHier%');
EndIf;

# Validate Subset
If( Trim( pSub ) @= '' );
    sMessage = 'No private subset specified';
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# No way to check if private subset exists with TurboIntegrator except via file system.
# Could include data directory param and concatenate with user, cube and view to check
# if private subset exists to handle error in the case that private sub does not exist

# Check for valid overwrite parameters
If( pOverwrite <> 0 & pOverwrite <> 1 );
    sMessage = 'Invalid overwrite existing public subset selection: ' | NumberToString( pOverwrite );
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( pOverwrite = 0 & HierarchySubsetExists( pDim, pHier, pSub ) = 1 );
    # If NOT overwriting current public subset AND subset of the same name already exists then cause minor error ( major error if not handled )
    sMessage = 'Public subset of same name already exists and Overwrite=0 specified';
    nErrors = nErrors + 1;
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

### Publish the subset ###
# PublishSubset publishes a named private subset on the server. This function was introduced in Planning Analytics 2.0.9.10/TM1 Server 11.8.9 and cannot be used in previous versions.
PublishSubset( sDimHier, pSub, pOverwrite );

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully published subset %pSub% in hierarchy %pDim%:%pHier% created by cient %pClient%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion