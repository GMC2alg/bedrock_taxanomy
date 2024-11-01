#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.viewandsubsets.delete', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pCube', '', 'pView', '', 'pSub', '', 'pMode', 1
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
# This process deletes a view and all subsets of the same name.

# Use case: 
# 1. In production environment used in Epilog to remove view & subsets used for processing.
# 2. In development/prototyping to manually clean up views & subsets. 

# Note:
# * Lists and wildcards are not supported in this process
# * A valid cube name pCube is mandatory otherwise the process will abort. 
# * A valid view name pView is mandatory otherwise the process will abort.
# * The matching assumption is based on **name**. Subsets of the same name as the view will be deleted (whether they were assigned to the view or not).
# * pMode 0 = Delete views and **indirectly** delete subsets via bedrock process call. If a subset cannot be deleted the process will continue and exit with minor error status.
# * pMode 1 = Delete views and **directly** delete subsets via SubsetDestroy function. If a subset cannot be deleted the process will abort with major error status.
# * pMode 2 = Delete views only and leave subsets as is.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName     = GetProcessName();
cUserName         = TM1User();
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cMsgInfoLevel     = 'INFO';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pView:%pView%, pSub:%pSub%, pMode:%pMode%.' ;  
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cTempSubset       = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
sMessage          = '';

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

### Validate Paramters ##
IF( pMode <> 0 & pMode <> 1 & pMode <> 2 );
    sMessage = 'Invailid mode value provided %pMode%. Do not destroy views or subsets.';
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ENDIF;

If( Trim( pCube ) @= '' );
    sMessage    = 'No cube specified.';
    nErrors     = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( pCube ) = 0 );    
    sMessage    = Expand( 'Invalid  cube specified: %pCube%.' );
    nErrors     = 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate pView 
If( Trim( pView ) @= '' );
    sMessage    = 'No view specified.';
    nErrors     = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( ViewExists(pCube, pView ) = 0 ); 
    sMessage    = Expand('There is no view :%pView% in %pCube% cube.') ;
    nErrors     = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Else;
    cView       = Trim( pView );
EndIf;

# Validate psubset
If( pSub @= '' );
    cSubset     = Trim( pView );
Else;
    cSubset     = Trim( pSub );
EndIf;

### Check for errors before continuing
If( nErrors     > 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

## Clean up view
ViewDestroy( pCube, cView );

## Clean up subsets
If( pMode       <= 1 );

    nDimCount = 0;
    i = 1;
    sDimName = TabDim( pCube, i );
    While( sDimName @<> '' );
        If( SubsetExists ( sDimName, cSubset ) = 1 );
            If( pMode = 0 );
                # "indirect" deletion
                 nRet = ExecuteProcess( '}bedrock.hier.sub.delete',
                  'pStrictErrorHandling', pStrictErrorHandling,
                	'pLogOutput', pLogOutput,
                	'pDim', sDimName,
                	'pHier','',
                	'pSub', cSubset,
                	'pDelim', If( Scan( '&', cSubset ) = 0, '&', ':' ),
                	'pMode', 0
                );
                If( pLogOutput >= 1 & nRet <> ProcessExitNormal() );
                    nErrors = nErrors + 1;
                    sMessage = 'Subset %cSubset% in dimension %sDimName% could not be deleted. It may be used in another view.';
                    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
                EndIf;
            ElseIf( pMode = 1 );
                # pMode=1, "direct" deletion
                SubsetDestroy( sDimName, cSubset );
            EndIf;
        EndIf;
        i = i + 1;
        sDimName = TabDim( pCube, i );
    End;

EndIf;
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully deleted views and subsets for cube  %pCube%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion