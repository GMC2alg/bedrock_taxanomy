#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.security.client.group.assign', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pClient', '', 'pGroup', '',
    	'pDelim','&', 'pAddOrRemove', 'Add', 'pSecurityRefresh', 'Yes'
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
# This process will assign Client and Group Security.

# Use case: Intended for production.
# 1/ After adding new user(s) using '}bedrock.security.client.create' they must be assigned to groups.

# Note:
# Naturally, valid  client (pClient) is mandatory otherwise the process can't make assignments:
# - This process can either add or remove clients to groups based on pAddOrRemove.
# - This process can work on a single client or multiple clients separated by a delimiter.
# - This process can work on a single group or multiple groups separated by a delimiter.
# - Therefore, it can be used to:
#   - Add/remove a single client to/from a single group.
#   - Add/remove many clients to/from a single group.
#   - Add/remove a single client to/from many groups.
#   - Add/remove many clients to/from many groups.
# - The caller can choose whether or not to refresh security at the end. Default is yes.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName = GetProcessName();
cTimeStamp = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt = NumberToString( INT( RAND( ) * 1000 ));
cUserName         = TM1User();
cTempSub          = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pClient:%pClient%, pGroup:%pGroup%, pDelim:%pDelim%, pAddOrRemove:%pAddOrRemove%, pSecurityRefresh:%pSecurityRefresh%.' ;  

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

# If no clients have been specified then terminate process
If( Trim( pClient ) @= '' );
    nErrors = 1;
    sMessage = 'No clients specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If no groups have been specified then terminate process
If( Trim( pGroup ) @= '' );
    nErrors = 1;
    sMessage = 'No groups specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If pSecurityRefresh is blank then default to yes
If( Trim( pSecurityRefresh ) @= '' );
    pSecurityRefresh = 'Yes';
EndIf;

# If pAddOrDelete has not been specified correctly then terminate process
If( Upper( pAddOrRemove ) @<> 'ADD' & Upper( pAddOrRemove ) @<> 'REMOVE' );
    nErrors = 1;
    sMessage = 'Incorrect value for pAddOrRemove: ' | pAddOrRemove | '. Valid values are Add or Remove';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If pSecurityRefresh has not been specified correctly then terminate process
If( Upper( pSecurityRefresh ) @<> 'YES' & Upper( pSecurityRefresh ) @<> 'NO' );
    nErrors = 1;
    sMessage = 'Incorrect value for pSecurityRefresh: ' | pSecurityRefresh | '. Valid values are Yes or No';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If blank delimiter specified then convert to default
If( pDelim @= '' );
  pDelim = '&';
EndIf;

### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### Add/remove clients to/from groups ###

# Loop through list of clients
sClients            = pClient;
nClientDelimIndex   = 1;
While( nClientDelimIndex <> 0 );
  nClientDelimIndex = Scan( pDelim, sClients );
  If( nClientDelimIndex = 0 );
    sClient         = sClients;
  Else;
    sClient         = Trim( SubSt( sClients, 1, nClientDelimIndex - 1 ) );
    sClients        = Trim( Subst( sClients, nClientDelimIndex + Long(pDelim), Long( sClients ) ) );
  EndIf;
  
  If(Scan('*', sClient) = 0 & Scan('?', sClient) = 0);
  # Don't attempt to process a blank client
   If( sClient @<> '' );
     # Check that client exists
     If( DimIx( '}Clients', sClient ) = 0 );
        nErrors     = 1;
        sMessage    = 'Client: ' | sClient | ' does not exists';
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
     ELSE;
        # Loop through list of Groups
        sGroups                 = pGroup;
        nGroupDelimIndex        = 1;
        While( nGroupDelimIndex <> 0 );
            nGroupDelimIndex    = Scan( pDelim, sGroups );
            If( nGroupDelimIndex = 0 );
                sGroup          = sGroups;
            Else;
                sGroup          = Trim( SubSt( sGroups, 1, nGroupDelimIndex - 1 ) );
                sGroups         = Trim( Subst( sGroups, nGroupDelimIndex + Long(pDelim), Long( sGroups ) ) );
            EndIf;
            If( Scan( '*', sGroup ) = 0 & Scan('?', sGroup ) = 0 );
                # Don't attempt to process a blank Group
                If( sGroup @<> '' );
                    # Check that group exists
                    If( DimIx( '}Groups', sGroup ) = 0 );
                        nErrors = 1;
                        sMessage = 'Group: ' | sGroup | ' does not exists';
                        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
                    ELSE;
                        # Add/Remove Client
                        If( Upper( pAddOrRemove ) @= 'ADD' );
                            AssignClientToGroup( sClient, sGroup );
                        ElseIf( Upper( pAddOrRemove ) @= 'REMOVE' );
                            RemoveClientFromGroup( sClient, sGroup );
                        EndIf;
                    EndIf;
                EndIf;
            ELSE;
                # Wildcard search string
                sGroup = '"'|sGroup|'"';
                sGroupDim = '}Groups';
                sMdx = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sGroupDim| '])},'| sGroup| ')}';
                If( SubsetExists( sGroupDim, cTempSub ) = 1 );
                    # If a delimited list of Groups includes wildcards then we may have to re-use the subset multiple times
                    SubsetMDXSet( sGroupDim, cTempSub, sMDX );
                Else;
                    # temp subset, therefore no need to destroy in epilog
                    SubsetCreatebyMDX( cTempSub, sMDX, sGroupDim, 1 );
                EndIf;
                nCountGrp = SubsetGetSize( sGroupDim, cTempSub );
                While( nCountGrp >= 1 );
                    sGroup = SubsetGetElementName( sGroupDim, cTempSub, nCountGrp );
                    # Add/Remove Client
                    If( Upper( pAddOrRemove ) @= 'ADD' );
                        AssignClientToGroup( sClient, sGroup );
                    ElseIf( Upper( pAddOrRemove ) @= 'REMOVE' );
                        RemoveClientFromGroup( sClient, sGroup );
                    EndIf;
                    nCountGrp = nCountGrp - 1;
                End;
            Endif;    
        End;
     EndIf;
   EndIf;
  ELSE;
  # Wildcard search string
        sClientDim = '}Clients';
        sClient = '"'|sClient|'"';
        sMdx = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sClientDim| '])},'| sClient| ')}';
        If( SubsetExists( sClientDim, cTempSub ) = 1 );
            # If a delimited list of Clients includes wildcards then we may have to re-use the subset multiple times
            SubsetMDXSet( sClientDim, cTempSub, sMDX );
        Else;
            # temp subset, therefore no need to destroy in epilog
            SubsetCreatebyMDX( cTempSub, sMDX, sClientDim, 1 );
        EndIf;
        # Loop through dimensions in clients with wildcard
        nCountClient = SubsetGetSize( '}Clients' , cTempSub );
        While( nCountClient >= 1 );
            
            sClient = SubsetGetElementName( '}Clients' , cTempSub, nCountClient );
            # Validate client name
            If( Dimix('}Clients', sClient) = 0 );
                nErrors = 1;
                sMessage = Expand( 'Client "%sClient%" does not exist.' );
                LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
            Else;
              # Loop through list of Groups
              sGroups = pGroup;
              nGroupDelimIndex = 1;
              While( nGroupDelimIndex <> 0 );
                nGroupDelimIndex = Scan( pDelim, sGroups );
                If( nGroupDelimIndex = 0 );
                   sGroup = sGroups;
                Else;
                   sGroup = Trim( SubSt( sGroups, 1, nGroupDelimIndex - 1 ) );
                   sGroups = Trim( Subst( sGroups, nGroupDelimIndex + Long(pDelim), Long( sGroups ) ) );
                EndIf;
                If( Scan( '*', sGroup ) = 0 & Scan('?',sGroup ) = 0);
                  # Don't attempt to process a blank Group
                  If( sGroup @<> '' );
                    # Check that group exists
                    If( DimIx( '}Groups', sGroup ) = 0 );
                        nErrors = 1;
                        sMessage = 'Group: ' | sGroup | ' does not exists';
                        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
                    ELSE;
                      # Add/Remove Client
                      If( Upper( pAddOrRemove ) @= 'ADD' );
                        AssignClientToGroup( sClient, sGroup );
                      ElseIf( Upper( pAddOrRemove ) @= 'REMOVE' );
                        RemoveClientFromGroup( sClient, sGroup );
                      EndIf;
                    EndIf;
                  EndIf;
                ELSE;
                # Wildcard search string
                sGroup = '"'|sGroup|'"';
                sGroupDim = '}Groups';
                sMdx = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sGroupDim| '])},'| sGroup| ')}';
                If( SubsetExists( sGroupDim, cTempSub ) = 1 );
                   # If a delimited list of objects includes wildcards then we may have to re-use the subset multiple times
                   SubsetMDXSet( sGroupDim, cTempSub, sMDX );
                Else;
                   # temp subset, therefore no need to destroy in epilog
                   SubsetCreatebyMDX( cTempSub, sMDX, sGroupDim, 1 );
                EndIf;
                nCountGrp = SubsetGetSize( sGroupDim, cTempSub );
                  While( nCountGrp >= 1 );
                      sGroup = SubsetGetElementName( sGroupDim, cTempSub, nCountGrp );
                      # Add/Remove Client
                      If( Upper( pAddOrRemove ) @= 'ADD' );
                          AssignClientToGroup( sClient, sGroup );
                      ElseIf( Upper( pAddOrRemove ) @= 'REMOVE' );
                          RemoveClientFromGroup( sClient, sGroup );
                      EndIf;
                      nCountGrp = nCountGrp - 1;
                  End;
               EndIf;
              End;
            Endif;
            nCountClient = nCountClient - 1;
        End;
  Endif;  
End;


### Refresh Security ###

If( Upper( pSecurityRefresh ) @= 'YES' );
  SecurityRefresh;
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully performed %pAddOrRemove% for client %pClient% to group %pGroup%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;


### End Epilog ###
#endregion