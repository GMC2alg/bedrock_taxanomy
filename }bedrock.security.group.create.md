#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.security.group.create', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pGroup', '', 'pAlias', '', 'pDelim', '&'
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
# This process will create client groups.

# Use case: Intended for development or production.
# 1/ Create initial security groups.
# 2/ Add security groups as business needs change.

# Note:
# Naturally, a group (pGroup) is mandatory otherwise the process will abort.
# - Multiple groups can be specified separated by a delimiter.
# - If group already exists then the process will skip that group.
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
cLogInfo            = 'Process:%cThisProcName% run with parameters pGroup:%pGroup%, pDelim:%pDelim%.' ;  

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors             = 0;
# If blank delimiter specified then convert to default
If( pDelim @= '' );
    pDelim          = '&';
EndIf;

# If no groups have been specified then terminate process
If( Trim( pGroup ) @= '' );
    nErrors         = 1;
    sMessage        = 'No groups specified';
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
    If( DimensionExists( '}ElementAttributes_}Groups' ) = 0 );
        AttrInsert( '}Groups', '', '}TM1_DefaultDisplayValue', 'A' );
    ElseIf( DimIx( '}ElementAttributes_}Groups', '}TM1_DefaultDisplayValue' ) = 0 );
        AttrInsert( '}Groups', '', '}TM1_DefaultDisplayValue', 'A' );
    EndIf;
EndIf;

### Split pGroups into individual groups and add ###
sGroups = pGroup;
nDelimiterIndex = 1;
While( nDelimiterIndex <> 0 );
    nDelimiterIndex = Scan( pDelim, sGroups );
    If( nDelimiterIndex = 0 );
        sGroup = sGroups;
    Else;
        sGroup = Trim( SubSt( sGroups, 1, nDelimiterIndex - 1 ) );
        sGroups = Trim( Subst( sGroups, nDelimiterIndex + Long(pDelim), Long( sGroups ) ) );
    EndIf;
    # Don't attempt to add a blank group
    If( sGroup @<> '' );
        If( DimIx( '}Groups', sGroup ) = 0 );
            AddGroup( sGroup );
        Else;
            #Skip group
        EndIf;
    EndIf;
End;

If( nErrors = 0 );
  DimensionSortOrder( '}Groups', 'ByName', 'Ascending', 'ByName' , 'Ascending' );
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

### Update Alias

If( nErrors = 0 );

  sAliases  = pAlias;
  sGroups   = pGroup;
  nDelimiterIndex = 1;

  While( nDelimiterIndex > 0 );
    nDelimiterIndex = Scan( pDelim, sAliases );
    If( nDelimiterIndex = 0 );
      sAlias    = sAliases;
    Else;
      sAlias    = Trim( SubSt( sAliases, 1, nDelimiterIndex - 1 ) );
      sAliases  = Trim( Subst( sAliases, nDelimiterIndex + Long(pDelim), Long( sAliases ) ) );
    EndIf;
    nDelimiterIndex = Scan( pDelim, sGroups );
    If( nDelimiterIndex = 0 );
      sGroup   = sGroups;
    Else;
      sGroup   = Trim( SubSt( sGroups, 1, nDelimiterIndex - 1 ) );
      sGroups  = Trim( Subst( sGroups, nDelimiterIndex + Long(pDelim), Long( sGroups ) ) );
    EndIf;
    
    If( DimIx( '}Groups', sGroup ) > 0 );
      If( sAlias @<> '' );
        AttrPutS( sAlias, '}Groups', sGroup, '}TM1_DefaultDisplayValue', 1 );
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully created Group %pGroup% .' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion