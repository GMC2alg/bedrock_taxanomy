#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess('}bedrock.server.encrypt.directory',
       'pLogOutput', pLogOutput,
       'pStrictErrorHandling', pStrictErrorHandling,
       'pType', pType,
       'pDirectory', pDirectory,
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
# This process this process unencrypts all files existing in a directory, using the tm1crypt utility

# Use case: To encrypts / unencrypts multile file in a directory. Calls sub-process.


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
cLogInfo            = 'Process:%cThisProcName% run with parameters pType:%pType%, pDirectory:%pDirectory%, pDestPath:%pDestPath%, pConfigLocation:%pConfigLocation%, pTM1CryptLocation:%pTM1CryptLocation%, pAction:%pAction%.' ;  
cMsgInfoContent     = 'User:%cUserName% Process:%cThisProcName% InfoMsg:%sMessage%';
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

## Validate the source directory
If ( pDirectory @= '' );
    nErrors         = 1;
    sMessage        = 'pDirectory is Blank';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;
sSourcePath = pDirectory;
If ( SubSt ( pDirectory, Long ( pDirectory ), 1 ) @<> sOSDelim );
  sSourcePath = sSourcePath | sOSDelim;
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

## Validate the dest path
sDestPath = pDestPath;
If ( pDestPath @= '' );
    sMessage        = 'pDestPath is Blank, using logging dir';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    sDestPath = GetProcessErrorFileDirectory;
EndIf;
If ( SubSt ( sDestPath, Long ( sDestPath ), 1 ) @<> sOSDelim );
  sDestPath = sDestPath | sOSDelim;
EndIf;

If ( sDestPath @= sSourcePath );
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

sFile = '';
sPrev = '';
sFile = WildcardFileSearch( sSourcePath | '****' | pType , sPrev);

While ( sFile @<> '' );
  IF( pLogoutput = 1 );
      sMessage = 'Processing file: ' | sFile;
      LogOutput('INFO', Expand( cMsgInfoContent ) );
  ENDIF;
  nRet = ExecuteProcess('}bedrock.server.encrypt.file',
     'pSourcePath', sSourcePath,
     'pSourceFile', sFile,
     'pDestPath', sDestPath,
     'pConfigLocation', pConfigLocation,
     'pTM1CryptLocation', pTM1CryptLocation,
     'pAction', pAction
    );
  If( nRet <> ProcessExitNormal() );
      nErrors = nErrors + 1;
      sMessage= 'Error in processing file: %sFile%.';
      DataSourceType = 'NULL';
      LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
  EndIf;
  sPrev = sFile;
  sFile = WildcardFileSearch( sSourcePath | '*' | pType , sPrev);
End;  
  
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully processed directory %pDirectory%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;
#endregion