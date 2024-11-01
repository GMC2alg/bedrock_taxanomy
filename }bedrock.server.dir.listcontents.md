#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.server.dir.listcontents', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pSrcDir', ''
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
# This process will list all files and processes in the data directory as text file output in the data directory itself.

# Use case: Intended for production.
# 1/ Obtain a listing of files in any folder.

# Note:
# The TI 1st creates a batch file that will list all the different type of objects seperately and then executes it.
# In some cases the path to the batch file (in the data directory) must be fully qualified.
# The batch file is deleted in Epilog.
#EndRegion @DOC

### TO DELETE - just included as temp measure so process compiles
pDebug = 0;
### TO DELETE - just included as temp measure so process compiles

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName       = GetProcessName();
cUserName           = TM1User();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pSrcDir:%pSrcDir%.' ;  
#cDebugFile = GetProcessErrorFileDirectory | cProcess | '.' | cTimeStamp | '.' | sRandomInt ;

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

If( sOS @= 'Linux' );
  cBatchFile = LOWER(cThisProcName) | '.sh';
Else;
  cBatchFile = cThisProcName | '.bat';
EndIf;

### Build Command ###
nErrors         = 0;

If( pSrcDir @= '' );
    # If no data directory provided then use current directory
    sCommand    = cBatchFile;
Else;
    # Trim off trailing backslash if present
    If( SubSt( pSrcDir, Long( pSrcDir ), 1 ) @= sOSDelim );
        pSrcDir = SubSt( pSrcDir, 1, Long( pSrcDir ) - 1 );
    EndIf;
    # Check that data directory exists
    If( FileExists( pSrcDir ) = 0 );
        nErrors  = 1;
        sMessage = 'Data directory does not exist: ' | pSrcDir;
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    Else;
        sCommand = pSrcDir | sOSDelim | cBatchFile;
    EndIf;
EndIf;

### Create Batch File ###
DatasourceASCIIQuoteCharacter='';
If( sOS @= 'Windows');
  ASCIIOUTPUT( sCommand, 'dir /b /s *.* > List_All_Data_Directory_Files.txt');
  ASCIIOUTPUT( sCommand, 'dir /b /s }*.* > List_All_Control_Objects.txt');
  ASCIIOUTPUT( sCommand, 'dir /b *.cub > List_All_Cubes.txt');
  
  ASCIIOUTPUT( sCommand, 'dir /b *.dim > List_All_Dimensions.txt');
  ASCIIOUTPUT( sCommand, 'dir /b *.pro > List_All_Processes.txt');
  ASCIIOUTPUT( sCommand, 'dir /b *.cho > List_All_Chores.txt');
  ASCIIOUTPUT( sCommand, 'dir /b *Bedrock*.pro > List_All_Bedrock_Processes.txt');
  
  ASCIIOUTPUT( sCommand, 'dir /b /s *.vue > List_All_Views.txt');
  ASCIIOUTPUT( sCommand, 'dir /b /s *.sub > List_All_Subsets.txt');
Else;
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f > List_All_Data_Directory_Files.txt ;');
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -name "}*" > List_All_Control_Objects.txt ;');
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -name "*.cub" > List_All_Cubes.txt ;');
  
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -name "*.dim" > List_All_Dimensions.txt ;');
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -name "*.pro" > List_All_Processes.txt ;');
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -name "*.cho" > List_All_Chores.txt ;');
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -iname "*Bedrock*.pro" > List_All_Bedrock_Processes.txt ;');
  
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -name "*.vue" > List_All_Views.txt ;');
  ASCIIOUTPUT( sCommand, 'find "$PWD" -type f -name "*.sub" > List_All_Subsets.txt ;');
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

### VALIDATION ###
# Check that batch file exists
If( FileExists( sCommand ) = 0 );
    nErrors = 1;
    sMessage = 'Batch file not found: ' | sCommand;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

### Execute Batch File ###
If( nErrors = 0 );
  If(sOS @= 'Windows');
    ExecuteCommand ( sCommand, 1 );
  Else;
    ExecuteCommand ( 'sh ' | sCommand, 1 );
  EndIf;
EndIf;

### Delete temporary batch file
ASCIIDelete ( sCommand );

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully listed contents from %pSrcDir% .' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion