#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.security.object.assign', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pGroup', '', 'pObjectType', '', 'pObject', '',
    	'pSecurityLevel', '', 'pSecurityRefresh', 'No', 'pDelim', '&'
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
# This process will assign security to specified object(s) for specified group(s). Both pGroup 
# and pObject parameters support delimited lists and wildcards.

# Use case: Intended for develpment or production.
# 1. Set up initial security assignments.
# 2. Update security assignments a business needs change.

# Note:
# Naturally, group (pGroup), object type (pObjectType) and security level (pSecurityLevel) are mandatory otherwise the process will abort:
# - Multiple groups can be specified separated by delimiter.
# Object names (pObject) can be listed with delimiters but need to be valid as well. 
# - Before using this process, you must first initialise object security.
# - For Applications, in Server Explorer go to Applications -> Security Assignments and assign an access security for an Application. 
# - For other object types follow a similar process but for that object type.
# - Then from the Server Explorer go to your ServerName -> Security -> Refresh Security.
# - Multiple objects can be specified separated by delimiter.
# - If groups and/or objects that don't exist are specified, then they will be skipped but valid groups and ojects will still be processed.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName       = GetProcessName();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cSecurityCube       = '}' | pObjectType | 'Security';
cTempSub            = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cUserName           = TM1User();
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pGroup:%pGroup%, pObjectType:%pObjectType%, pObject:%pObject%, pSecurityLevel:%pSecurityLevel%, pSecurityRefresh:%pSecurityRefresh%, pDelim:%pDelim%.' ;  

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors             = 0;
# If no groups have been specified then terminate process
If( Trim( pGroup ) @= '' );
    nErrors         = 1;
    sMessage        = 'No groups specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate Object Type
If( pObjectType @<> 'Application' & pObjectType @<> 'Cube' & pObjectType @<> 'Dimension' & pObjectType @<> 'Process' & pObjectType @<> 'Chore' );
    nErrors         = 1;
    sMessage        = 'Invalid object type specified: ' | pObjectType | '. Valid object types are: Application, Cube, Dimension, Process and Chore';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If no objects have been specified then terminate process
If( Trim( pObject ) @= '' );
    nErrors         = 1;
    sMessage        = 'No objects specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate that object type has object security enabled
If( CubeExists( cSecurityCube ) = 0 );
    sObjectTypePlural = pObjectType;
    If( pObjectType @= 'Process');
        sObjectTypePlural = sObjectTypePlural | 'es';
    Else;
        sObjectTypePlural = sObjectTypePlural | 's';
    EndIf;
    nErrors = 1;
    sMessage = 'You must first initialise ' | pObjectType | ' Security. In the Server Explorer go to ' |
               sObjectTypePlural | ' -> Security Assignments, and assign an access security to a ' |
               pObjectType | '. Then from the Server Explorer go to your ServerName -> Security -> Refresh Security';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Set Object Type Dimension
sObjectTypeDimension = '}' | pObjectType;
If( pObjectType @= 'Application' );
    sObjectTypeDimension = sObjectTypeDimension | 'Entries';
ElseIf( pObjectType @= 'Process' );
    sObjectTypeDimension = sObjectTypeDimension | 'es';
Else;
    sObjectTypeDimension = sObjectTypeDimension | 's';
EndIf;

# Validate Security Level
# Allowable Security Levels by Object Type (excluding RESERVE and LOCK):
#   Cube        : NONE, READ, ADMIN, WRITE
#   Dimension   : NONE, READ, ADMIN, WRITE
#   Application : NONE, READ, ADMIN
#   Process     : NONE, READ
#   Chore       : NONE, READ
pSecurityLevel = Upper( pSecurityLevel );
If( pSecurityLevel @= 'NONE' % pSecurityLevel @= 'READ' % 
  ( pSecurityLevel @= 'ADMIN' & ( pObjectType @= 'Cube' % pObjectType @= 'Dimension' % pObjectType @= 'Application' ) ) %
  ( pSecurityLevel @= 'WRITE' & ( pObjectType @= 'Cube' % pObjectType @= 'Dimension' ) ) );
  # Valid Security Levels
Else;
    nErrors = 1;
    sMessage = 'Invalid security level parameter: ' | pSecurityLevel | ' for object type: ' | pObjectType;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If pSecurityRefresh is blank then default to yes
If( Trim( pSecurityRefresh ) @= '' );
    pSecurityRefresh = 'Yes';
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

### Assign Application Security ###
# Loop through list of groups
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
  

 If(Scan('*',sGroup) = 0);
   
 
  # Don't attempt to process a blank group or ADMIN
  If( sGroup @<> '' & sGroup @<> 'ADMIN' );
    # Check that Group exists
    If( DimIx( '}Groups', sGroup ) = 0 );
        nErrors = 1;
        sMessage = 'Group: ' | sGroup | ' does not exists';
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
     ELSE;

      # Loop through list of objects
      sObjects = pObject;
      nObjectDelimIndex = 1;

      While( nObjectDelimIndex <> 0 );
        nObjectDelimIndex = Scan( pDelim, sObjects );
        If( nObjectDelimIndex = 0 );
          sObject = sObjects;
        Else;
          sObject = Trim( SubSt( sObjects, 1, nObjectDelimIndex - 1 ) );
          sObjects = Trim( Subst( sObjects, nObjectDelimIndex + Long(pDelim), Long( sObjects ) ) );
        EndIf;
        If( Scan( '*', sObject ) = 0);
         # Don't attempt to process a blank object
         If( sObject @<> '' );
           # Check that object exists
           If( DimIx( sObjectTypeDimension, sObject ) = 0 );
                nErrors = 1;
                sMessage = 'Object: ' | sObject | ' does not exists';
                LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
          ELSE;
             # Assign Security
             If( CellIsUpdateable( cSecurityCube, sObject, sGroup ) = 1 );
               If( nErrors = 0);
                 CellPutS( pSecurityLevel, cSecurityCube, sObject, sGroup );
               EndIf;
             EndIf;
           EndIf;
         EndIf;
        ELSE;
          # Wildcard search string
         sObject = '"'|sObject|'"';
         sMdx = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sObjectTypeDimension| '])},'| sObject| ')}';
         If( SubsetExists( sObjectTypeDimension, cTempSub ) = 1 );
            # If a delimited list of objects includes wildcards then we may have to re-use the subset multiple times
            SubsetMDXSet( sObjectTypeDimension, cTempSub, sMDX );
         Else;
            # temp subset, therefore no need to destroy in epilog
            SubsetCreatebyMDX( cTempSub, sMDX, sObjectTypeDimension, 1 );
         EndIf;
         nCountObj = SubsetGetSize( sObjectTypeDimension, cTempSub );
         While( nCountObj >= 1 );
            sObject = SubsetGetElementName( sObjectTypeDimension, cTempSub, nCountObj );
            # Assign Security
             If( CellIsUpdateable( cSecurityCube, sObject, sGroup ) = 1 );
               If( nErrors = 0);
                 CellPutS( pSecurityLevel, cSecurityCube, sObject, sGroup );
               EndIf;
             EndIf;
            nCountObj = nCountObj - 1;
         End;
        EndIf;
      End;
    EndIf;
  EndIf;
 ELSE;
# Wildcard search string
        sGroupDim = '}Groups';
        sGroup = '"'|sGroup|'"';
        sMdx = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sGroupDim| '])},'| sGroup| ')}';
        If( SubsetExists( sGroupDim, cTempSub ) = 1 );
            # If a delimited list of groups includes wildcards then we may have to re-use the subset multiple times
            SubsetMDXSet( sGroupDim, cTempSub, sMDX );
        Else;
            # temp subset, therefore no need to destroy in epilog
            SubsetCreatebyMDX( cTempSub, sMDX, sGroupDim, 1 );
        EndIf;
        # Loop through dimensions in groups with wildcard
        nCountGroup = SubsetGetSize( '}Groups' , cTempSub );
        While( nCountGroup >= 1 );
            
            sGroup = SubsetGetElementName( '}Groups' , cTempSub, nCountGroup );
            # Validate group name
            If( Dimix('}Groups', sGroup) = 0 );
                nErrors = 1;
                sMessage = Expand( 'Group "%sGroup%" does not exist.' );
                LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
            Else;
              # Loop through list of objects
              sObjects = pObject;
              nObjectDelimIndex = 1;

              While( nObjectDelimIndex <> 0 );
                nObjectDelimIndex = Scan( pDelim, sObjects );
                If( nObjectDelimIndex = 0 );
                  sObject = sObjects;
                Else;
                  sObject = Trim( SubSt( sObjects, 1, nObjectDelimIndex - 1 ) );
                  sObjects = Trim( Subst( sObjects, nObjectDelimIndex + Long(pDelim), Long( sObjects ) ) );
                EndIf;
                If( Scan( '*', sObject ) = 0);
                # Don't attempt to process a blank object
                  If( sObject @<> '' );
                    # Check that object exists
                    If( DimIx( sObjectTypeDimension, sObject ) = 0 );
                        nErrors = 1;
                        sMessage = 'Object: ' | sObject | ' does not exists';
                        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
                    ELSE;
                      # Assign Security
                      If( CellIsUpdateable( cSecurityCube, sObject, sGroup ) = 1 );
                        If( nErrors = 0);
                          CellPutS( pSecurityLevel, cSecurityCube, sObject, sGroup );
                        EndIf;
                      EndIf;
                    EndIf;
                  EndIf;
                ELSE;
                # Wildcard search string
                sObject = '"'|sObject|'"';
                sMdx = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sObjectTypeDimension| '])},'| sObject| ')}';
                If( SubsetExists( sObjectTypeDimension, cTempSub ) = 1 );
                    # If a delimited list of objects includes wildcards then we may have to re-use the subset multiple times
                    SubsetMDXSet( sObjectTypeDimension, cTempSub, sMDX );
                Else;
                    # temp subset, therefore no need to destroy in epilog
                    SubsetCreatebyMDX( cTempSub, sMDX, sObjectTypeDimension, 1 );
                EndIf;
                  nCountObj = SubsetGetSize( sObjectTypeDimension, cTempSub );
                  While( nCountObj >= 1 );
                      sObject = SubsetGetElementName( sObjectTypeDimension, cTempSub, nCountObj );
                      # Assign Security
                      If( CellIsUpdateable( cSecurityCube, sObject, sGroup ) = 1 );
                        If( nErrors = 0);
                          CellPutS( pSecurityLevel, cSecurityCube, sObject, sGroup );
                        EndIf;
                      EndIf;
                      nCountObj = nCountObj - 1;
                  End;
                EndIf;
              End;
            Endif;
            nCountGroup = nCountGroup - 1;
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully assigned object %pObject% of type %pObjectType% security level %pSecurityLevel% for group %pGroup%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion