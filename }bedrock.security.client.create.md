#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.security.client.create', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pClient', '', 'pAlias', '', 'pPassword', '', 'pDelim', '&' 
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
# This process will create clients, assign a password and max ports.

# Use case: Intended for production.
# 1/ Create clients for multiple new hires simultaneously.

# Note:
# Naturally, a client (pClient) is mandatory otherwise the process will abort.
# - Multiple clients can be specified separated by a delimiter.
# - If client already exists then the process will not attempt to re-create it but will reset password and max ports.
# - Each client will have to be assigned to a group afterwards.
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
cLogInfo            = 'Process:%cThisProcName% run with parameters pClient:%pClient%, pPassword:******, pDelim:%pDelim%.' ;  

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors             = 0;

# If blank delimiter specified then convert to default
If( pDelim @= '' );
  pDelim            = '&';
EndIf;

# If no clients have been specified then terminate process
If( Trim( pClient ) @= '' );
  nErrors           = 1;
  sMessage          = 'No clients specified.';
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

# Alias
If( pAlias @<> '' );
    If( DimensionExists( '}ElementAttributes_}Clients' ) = 0 );
        AttrInsert( '}Clients', '', '}TM1_DefaultDisplayValue', 'A' );
    ElseIf( DimIx( '}ElementAttributes_}Clients', '}TM1_DefaultDisplayValue' ) = 0 );
        AttrInsert( '}Clients', '', '}TM1_DefaultDisplayValue', 'A' );
    EndIf;
EndIf;

### Split pClient into individual Clients and add ###
sClients            = pClient;
nDelimiterIndex     = 1;
While( nDelimiterIndex <> 0 );
    nDelimiterIndex   = Scan( pDelim, sClients );
    If( nDelimiterIndex = 0 );
        sClient         = sClients;
    Else;
        sClient         = Trim( SubSt( sClients, 1, nDelimiterIndex - 1 ) );
        sClients        = Trim( Subst( sClients, nDelimiterIndex + Long(pDelim), Long( sClients ) ) );
    EndIf;
    # Don't attempt to add a blank client
    If( sClient @<> '' );
        If( DimIx( '}Clients', sClient ) = 0 );
            If( nErrors = 0 );
                AddClient( sClient );
            EndIf;
        EndIf;
    EndIf;
End;

If( nErrors = 0 );
    DimensionSortOrder( '}Clients', 'ByName', 'Ascending', 'ByName' , 'Ascending' );
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


### Update password & Alias

If( nErrors = 0 );

  sAliases  = pAlias;
  sClients  = pClient;
  nDelimiterIndex = 1;

  While( nDelimiterIndex > 0 );
    nDelimiterIndex = Scan( pDelim, sAliases );
    If( nDelimiterIndex = 0 );
      sAlias    = sAliases;
    Else;
      sAlias    = Trim( SubSt( sAliases, 1, nDelimiterIndex - 1 ) );
      sAliases  = Trim( Subst( sAliases, nDelimiterIndex + Long(pDelim), Long( sAliases ) ) );
    EndIf;
    nDelimiterIndex = Scan( pDelim, sClients );
    If( nDelimiterIndex = 0 );
      sClient   = sClients;
    Else;
      sClient   = Trim( SubSt( sClients, 1, nDelimiterIndex - 1 ) );
      sClients  = Trim( Subst( sClients, nDelimiterIndex + Long(pDelim), Long( sClients ) ) );
    EndIf;
    
    If( DimIx( '}Clients', sClient ) > 0 );
      AssignClientPassword( sClient, pPassword );
      If( sAlias @<> '' );
        AttrPutS( sAlias, '}Clients', sClient, '}TM1_DefaultDisplayValue', 1 );
      EndIf;
    EndIf;
  End;

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully added user(s) %pClient% to }Clients Dimension.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion