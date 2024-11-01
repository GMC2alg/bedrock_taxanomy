#region Prolog
#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

#Region @DOC
# Description:
# This process will run the TI ExecuteCommand function and print the output to Server Logs.

# Use case: Intended for production.
# 1/ To run an ExecuteCommand function from any part of the model, including RushTI or third party system without direct access to TI Editor.
# 2/ To remove the requirement of creating a one off process to use this function
# 3/ To compress/uncompress files
# 4/ To copy files and folders from the TM1 server
# 5/ To delete files and folders from the TM1 server
# 6/ To list and kill tasks running in the TM1 server
# 7/ To export and import registry keys such as ODBC data sources

#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName = GetProcessName();
cUser = TM1User();
cUserName = '';
If( DimIx('}Clients', cUser) > 0 );
    cUserName = AttrS('}Clients', cUser, '}TM1_DefaultDisplayValue');
EndIf;
cUserName = IF( cUserName @<> '', cUserName, 'ADMIN' );
cMsgErrorLevel = 'ERROR';
cMsgErrorContent = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cCmdOutputDir = GetProcessErrorFileDirectory;
cCmdOutputFile = cCmdOutputDir | GetProcessName() | '.txt';

## LogOutput parameters
If( pLogOutput = 1 );
  sLogInfo = Expand('Process "%cThisProcName%" run with parameters: pCommand: "%pCommand%", pWait: %pWait%, pPowerShell: %pPowerShell%'); 
  LogOutput ( 'INFO', sLogInfo );
  nStart = Now();
EndIf;

### Validate Parameters ###
nErrors = 0;
If ( pCommand @= '' );
  sMessage = 'parameter pCommand is blank';
  LogOutput ( 'ERROR', sMessage );
  ProcessQuit;
EndIf;

### ExecuteCommand ###

# Check if the pCommand parameter is enclosed in quotes and remove it if it is
If( Subst(pCommand, 1, 1) @= '"' );
  sCommand = Delet(pCommand, 1, 1);
  sCommand = Delet(sCommand, Long(sCommand), 1);
Else;
  sCommand = pCommand;
EndIf;

If( pPowerShell = 1 );
  #Prepare the full command for Powershell
  sCommand = 'POWERSHELL.EXE -Command "& {' | pCommand | '}" 1> ' | cCmdOutputFile | ' 2>&1';
Else;
  #Prepare the full command for Windows CMD
  sCommand = 'CMD.EXE /C "' | sCommand | '" 1> ' | cCmdOutputFile | ' 2>&1';
EndIf;

#Execute the command in the TM1 server
ExecuteCommand ( sCommand, pWait );

#If pLogOutput is true then define the command output file as data source
If( pLogOutput = 1 );
  DataSourceType = 'CHARACTERDELIMITED';
  DatasourceNameForServer = cCmdOutputFile;
EndIf;
#endregion
#region Metadata
#****Begin: Generated Statements***
#****End: Generated Statements****
#endregion
#region Data
#****Begin: Generated Statements***
#****End: Generated Statements****

# Write the command output to Server Logs
sLogInfo = Expand('Process "%cThisProcName%": %vCommandOutput%');
LogOutput( 'INFO', sLogInfo);
#endregion
#region Epilog
#****Begin: Generated Statements***
#****End: Generated Statements****

### LogOutput ###

If( pLogOutput = 1 );
    sSec     = NumberToStringEx( 86400*(Now() - nStart),'#,##0.0', '.', ',' );
    sLogInfo = Expand('Process "%cThisProcName%" completed in %sSec% seconds.'); 
    LogOutput( 'INFO', sLogInfo );
EndIf;

### Return code & final error message handling
If( nErrors > 0 );
    sMessage = 'the process incurred at least 1 error. Please see above lines in this file for more details.';
    nProcessReturnCode = 0;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    sProcessReturnCode = Expand( '%sProcessReturnCode% Process "%cThisProcName%" completed with errors. Check tm1server.log for details.' );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    EndIf;
Else;
    sProcessAction = Expand( 'Process "%cThisProcName%" completed successfully.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### Optional: Clean the command output file
#sCommand = 'CMD.EXE /C "TYPE NUL > "' | cCmdOutputFile | '" "';
#ExecuteCommand( sCommand, 0 );

### End Epilog ###
#endregion