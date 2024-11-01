#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.security.cube.cellsecurity.destroy', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pCube', '', 'pDelim', '&'
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
# This process will destroy the cell security cube(s) for the specified cube(s).

# Use case: Intended for development.
# 1/ Remove cell level security for one or more cubes.

# Note:
# Naturally, a valid cube (pCube) is mandatory otherwise the process will abort.
# If the cube does not have cell security set up, it will skip that cube but log an error.
# Multiple cubes can be specified separated by the pDelim or by using wildcards (*).
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName = GetProcessName();
cTimeStamp = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt = NumberToString( INT( RAND( ) * 1000 ));
DatasourceASCIIQuoteCharacter = '';
cUserName         = TM1User();
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% Message:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pDelim:%pDelim%.' ;  

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

## check operating system
If( SubSt( GetProcessErrorFileDirectory, 2, 1 ) @= ':' );
  sOS = 'Windows';
  sOSDelim = '\';
ElseIf( Scan( '/', GetProcessErrorFileDirectory ) > 0 );
  sOS = 'Linux';
  sOSDelim = '/';
Else;
  sOS = 'Windows';
  sOSDelim = '\';
EndIf;

### Validate Parameters ###
nErrors = 0;

# If blank delimiter specified then convert to default
If( pDelim @= '' );
    pDelim = '&';
EndIf;

# If no cubes have been specified then terminate process
If( Trim( pCube ) @= '' );
    nErrors = 1;
    sMessage = 'No cube(s) specified.';
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

### Split pCubes into individual Cubes  ###
sCubes                  = pCube;
nDelimiterIndex         = 1;
While( nDelimiterIndex <> 0 );
    nDelimiterIndex     = Scan( pDelim, sCubes );
    If( nDelimiterIndex = 0 );
        sCube           = sCubes;
    Else;
        sCube           = Trim( SubSt( sCubes, 1, nDelimiterIndex - 1 ) );
        sCubes          = Trim( Subst( sCubes, nDelimiterIndex + Long(pDelim), Long( sCubes ) ) );
    EndIf;
  
    # Check if a wildcard has been used to specify the Cube name.
    # If it hasn't then just delete the Cube if it exists
    # If it has then search the relevant Cube folder to find the matches
    If( Scan( '*', sCube ) = 0 );
        If( CubeExists( sCube ) = 1 ); 
            If(CubeExists( '}CellSecurity_' | sCube ) = 1);
                nRet = CellSecurityCubeDestroy( sCube );
                If( nRet = 1 );
                    sMessage = '}CellSecurity_' | sCube | ' successfully destroyed.';
                    LogOutput( 'INFO', Expand( cMsgErrorContent ) );
                Else;
                    nErrors = 1;
                    sMessage = 'Error. Could not destroy }CellSecurity_' | sCube;
                    LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
                EndIf;
            Endif;
        Endif;
    Else;
        # Wildcard search string
        sSearch                     = Expand('.%sOSDelim%%sCube%.cub');

        # Find all Cubes that match search string
        sFilename                   = WildcardFileSearch( sSearch, '' );
        While( sFilename @<> '' );
            # Trim .cub off the filename
            sCube                   = SubSt( sFilename, 1, Long( sFilename ) - 4 );
            # Destroy Cube
            If( CubeExists( sCube ) = 1 ); 
                If(CubeExists( '}CellSecurity_' | sCube ) = 1);
                    nRet            = CellSecurityCubeDestroy( sCube );
                    If( nRet = 1 );
                        sMessage    = '}CellSecurity_' | sCube | ' successfully destroyed.';
                        LogOutput( 'INFO', Expand( cMsgErrorContent ) );
                    Else;
                        nErrors     = 1;
                        sMessage    = 'Error. Could not destroy }CellSecurity_' | sCube;
                        LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
                    EndIf;
                Endif;
            Endif;
            sFilename               = WildcardFileSearch( sSearch, sFilename );
        End;
    EndIf;

End;

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully detsroyed cell security for cube  %pCube%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

sProcessReturnCode = pCube;

### End Epilog ###
#endregion