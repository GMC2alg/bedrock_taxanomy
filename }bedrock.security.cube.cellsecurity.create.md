#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.security.cube.cellsecurity.create', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pCube', '', 'pDim', ''
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
# This process will create a cell security cube for the specified cube for the specified list of dimensions 
# using the TI function _CellSecurityCubeCreate_. The benefit of this process is not needing to write a custom
# process each time in order to create a cell security cube.

# Use case: Intended for development.
# 1/ Set up cell security cubes

# Note:
# * Naturally, a valid cube (pCube) is mandatory otherwise the process will abort.
# * If cell security has already been set up the TI will abort.
# * The pDim parameter must map _ALL_ the dimensions in order in the cube with a 0 or 1.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName       = GetProcessName();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cUserName           = TM1User();
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% Message:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pDim:%pDim%.' ;  
cDelim              = ':';

## LogOutput parameters
IF( pLogOutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

# If no cube has been specified then terminate process
If( Trim( pCube ) @= '' );
    nErrors = 1;
    sMessage = 'No cube specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( pCube ) = 0 );
    nErrors = 1;
    sMessage = Expand('Cube %pCube% does not exist.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Check if cell security cube already exists
If( CubeExists(  '}CellSecurity_' | pCube ) = 1 );
    nErrors = 1;
    sMessage = 'Cell Security cube already exists.';
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

### Count dimensions in cube
nDims               = CubeDimensionCountGet( pCube );

### If pDim is wildcard, then = ALL (no restrictions on dimensions)
pDim = Trim( pDim );
If( pDim @= '*' );
   pDim = Fill( '1:', 2 * nDims - 1 );
EndIf;
### Count dimensions mapped in pDim ###
sDimensions         = pDim;
nDelimiterIndex     = 1;
nMapDims            = 0;
iDim                = 1;
While( nDelimiterIndex <> 0 );
    nMapDims        = iDim;
    nDelimiterIndex = Scan( cDelim, sDimensions );
    If( nDelimiterIndex = 0 );
        sDimension  = sDimensions;
    Else;
        sDimension  = Trim( SubSt( sDimensions, 1, nDelimiterIndex - 1 ) );
        sDimensions = Trim( Subst( sDimensions, nDelimiterIndex + Long(cDelim), Long( sDimensions ) ) );
    EndIf;
    # Redundant?
    If( sDimension @= '1' );
        sMessage    = ' INCLUDE in cell security cube';
    ElseIf( sDimension @= '0' );
        sMessage    = ' EXCLUDE from cell security cube';
    Else;
        sMessage    = ' INVALID map parameter: ' | sDimension;
    EndIF;
    iDim            = iDim + 1;
End;

### Check dimension count of dimension map vs. dimensions in cube ###
If( nDims <> nMapDims );
    nErrors         = 1;
    sMessage        = 'Parameter count of dimension map does not match dimension count of cube!';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Else;
    nRet            = CellSecurityCubeCreate ( pCube, pDim );
    If( nRet = 1 );
        sMessage    = '}CellSecurity_' | pCube | ' successfully created';
        LogOutput( 'INFO', Expand( cMsgErrorContent ) );
    Else;
        sMessage    = 'Error. Could not create }CellSecurity_' | pCube;
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully created cell security for %pCube% and %pDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion