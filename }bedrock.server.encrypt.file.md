#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess('}bedrock.server.encrypt.file', 'pLogOutput', pLogOutput,
       'pStrictErrorHandling', pStrictErrorHandling,
       'pSourcePath', pSourcePath,
       'pSourceFile', pSourceFile,
       'pDestPath', pDestPath,
       'pConfigLocation', pConfigLocation,
       'pTM1CryptLocation', pTM1CryptLocation,
       'pAction', pAction
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
# This process this process encrypts / unencrypts a file using the tm1crypt utility

# Use case: To encrypts / unencrypts a single file


# Note: Generated commands will only work when the TM1 isntance is entrypted
# 
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
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pSourcePath:%pSourcePath%, pSourceFile:%pSourceFile%, pDestPath:%pDestPath%, pConfigLocation:%pConfigLocation%, pTM1CryptLocation:%pTM1CryptLocation%, pAction:%pAction%.' ;  
cMsgInfoContent    = 'User:%cUserName% Process:%cThisProcName% InfoMsg:%sMessage%';
nDataCount        = 0;
nErrors           = 0;

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###

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

## Validate the action
sAction = '';
If ( pAction @= '4' % pAction @= '5');
    sAction = pAction;
ELSE;
    nErrors         = 1;
    sMessage        = 'Specified Action is not valid';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate config and exe
If ( FileExists( pConfigLocation ) = 0 );
    nErrors         = 1;
    sMessage        = 'Specified config file not found';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;  

If ( FileExists( pTM1CryptLocation ) = 0 );
    nErrors         = 1;
    sMessage        = 'Specified tm1crypt file not found';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;  

## Validate the source file/path and dest file/path
If ( pSourcePath @= '' );
    nErrors         = 1;
    sMessage        = 'pSourcePath is Blank';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;
sSourcePath = pSourcePath;
If ( SubSt ( pSourcePath, Long ( pSourcePath ), 1 ) @<> sOSDelim );
  sSourcePath = sSourcePath | sOSDelim;
EndIf;

sDestPath = pDestPath;
If ( pDestPath @= '' );
    sMessage        = 'pDestPath is Blank, using logging dir';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    sDestPath = GetProcessErrorFileDirectory;
EndIf;
If ( SubSt ( sDestPath, Long ( sDestPath ), 1 ) @<> sOSDelim );
  sDestPath = sDestPath | sOSDelim;
EndIf;

If ( pSourceFile @= '' );
    nErrors         = 1;
    sMessage        = 'pSourceFile is Blank';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;
sFileSrc = sSourcePath | pSourceFile;

If ( FileExists( sFileSrc ) = 0 );
    nErrors         = 1;
    sMessage        = '%sFileSrc% not found';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;  

sFileDest = sDestPath;
If ( SubSt ( sFileDest, Long ( sFileDest ), 1 ) @= sOSDelim );
  sFileDest = SubSt ( sFileDest, 1, Long ( sFileDest ) - 1 );
EndIf;

If ( sFileDest @= sSourcePath );
    nErrors         = 1;
    sMessage        = 'Destination is the same as source';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;  

### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

## Create contents of the bat file

DatasourceASCIIQuoteCharacter= '';

sBat = '"' | pTM1CryptLocation | '" -i "' | pConfigLocation | '" -action ' | sAction | ' -filesrc "' | sFileSrc | '" -filedest "' | sFileDest |'"';
If( sOS @= 'Windows' );
    sBatFile = sDestPath | pSourceFile | '.bat';
Else;
    sBatFile = sDestPath | pSourceFile | '.sh';
EndIf;
ASCIIOutput( sBatFile, sBat);
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

### Run that bat file

If( nErrors = 0 );
  If(sOS @= 'Windows');
    ExecuteCommand( sBatfile, 1 );
  Else;
    ExecuteCommand ( 'sh ' | sBatfile, 1 );
  EndIf;
  ASCIIDelete( sBatFile );
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully processed file %pSourceFile%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;
#endregion