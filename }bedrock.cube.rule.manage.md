#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.rule.manage', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pCube', '', 'pMode', '',
    	'pFileName', '', 'pDelim','&', 
    	'pPath', ''
	);
EndIf;
#EndRegion CallThisProcess

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# ####################
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

#Region @DOC
# Description:
# This process will backup & remove **OR** re-attach the rule file to cube.

# Use case: Intended to be used in production.
# 1/ Remove rule file before data load to speed up data load.
# 2/ Re-attach rule file after data load.

# Note:
# Naturally, a valid cube name (pCube) is mandatory otherwise the process will abort.
# The mandatory pMode parameter must be set to 'Unload' to remove/unload the rule file.
# The pMode parameter must be set to 'Load' to re-attach the rule file and apply the rules.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###

cThisProcName       = GetProcessName();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cTempSub            = cThisProcName | '_' | cTimeStamp | '_' | cRandomInt;
cCubeDim            = '}Cubes';
cCubeHier           = cCubeDim;
sPath               = '';
cUserName           = TM1User();
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pMode:%pMode%, pFileName:%pFileName%, pDelim:%pDelim%, pPath:%pPath%.' ;  
cDimCubes           = '}Cubes';

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

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

nErrors = 0;

### PROCESS PROPERTIES
DatasourceASCIIDelimiter = '';
DatasourceASCIIQuoteCharacter = '';

##Validate Mode
If(upper(pMode) @<> 'LOAD' & upper(pMode) @<> 'UNLOAD');
    sMessage = Expand('Invalid Mode: %pMode%. Valid Modes are Load or Unload');
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;

##Validate Cube
If( Trim(pCube) @= '' );
    sMessage = Expand('No cube specified');
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;

## Default filter delimiters
If( pDelim     @= '' );
    pDelim     = '&';
EndIf;

# define backupdir
If(pPath @<> '');
    If(Subst(pPath,long(pPath),1) @= sOSDelim );
        sPath = pPath;
    Else;
        sPath = pPath | sOSDelim;
    Endif;
Else;
  sPath = '.' | sOSDelim;
Endif;

## Default files names for storing rule and backups
IF(pFileName@='');
  sRuleFileName         = '%sCube%.txt';
  sBackupFileName       = '%sCube%.rux.bkp_%cTimeStamp%.txt';
Else;
  sRuleFileName         = pFileName;
  sBackupFileName       = '%pFileName%.bkp_%cTimeStamp%.txt';
Endif;  
  
# Loop through list of Cubes
sCubes = pCube;
nCubeDelimIndex = 1;

While( nCubeDelimIndex <> 0 );
  nCubeDelimIndex = Scan( pDelim, sCubes );
  If( nCubeDelimIndex = 0 );
    sCube = sCubes;
  Else;
    If (sOS @= 'Windows');
      sCube = Trim( SubSt( sCubes, 1, nCubeDelimIndex - 1 ) );
    Else;
      sCube = LOWER(Trim( SubSt( sCubes, 1, nCubeDelimIndex - 1 ) ));
    EndIf;
    sCubes = Trim( Subst( sCubes, nCubeDelimIndex + Long(pDelim), Long( sCubes ) ) );
  EndIf;

  sMDX = Expand( '{TM1FILTERBYPATTERN(TM1SUBSETALL([}Cubes]), "%sCube%")}' );

  sProc = '}bedrock.hier.sub.create.bymdx';
  ExecuteProcess( sProc,
    'pLogOutput', pLogOutput,
    'pStrictErrorHandling', pStrictErrorHandling,
    'pDim', cDimCubes,
    'pHier', '',
    'pSub', cTempSub,
    'pMDXExpr', sMDX,
    'pConvertToStatic', 1,
    'pTemp', 1
  );

  nCube = 1;
  nCubes = SubsetGetSize( cDimCubes, cTempSub );
  While( nCube <= nCubes );
    sCube = SubsetGetElementName( cDimCubes, cTempSub, nCube );
    nCube = nCube + 1;
    
    If( sOS @= 'Windows');
      cCubeRuleFileName = '.' | sOSDelim | sCube | '.rux';
    Else;
      cCubeRuleFileName = '.' | sOSDelim | LOWER(sCube) | '.rux';
    EndIf;
    If( sOS @= 'Windows');
      cStoreDirFile = sPath | Expand(sRuleFileName);
      cBackupDirFile = sPath | Expand(sBackupFileName); 
    Else;
      cStoreDirFile = sPath | LOWER(Expand(sRuleFileName));
      cBackupDirFile = sPath | LOWER(Expand(sBackupFileName)); 
    EndIf;
    # if there already is a rule file
    If(FileExists(cCubeRuleFileName) <> 0);
      ##Loading the Rule###
      If(Upper(pMode) @= 'LOAD');
        ##Backup the existing rule (saved as .bkp.txt in given path or data directory) 
        ##and load the new rule file. New Rule file should available in the given path or in data directory
        If( sOS @= 'Windows');
          sCmd = 'cmd /c "copy """' | cCubeRuleFileName | '"""  """' | cBackupDirFile |'""" "';          
        Else;
          sCmd = 'cp "' | cCubeRuleFileName | '" "' | cBackupDirFile | '"';        
        EndIf;
        ExecuteCommand(sCmd,1);
        ## Check if the backup file exists
        If( FileExists( cBackupDirFile ) <> 0 );
          RuleLoadFromFile( sCube, cStoreDirFile);
        Else;
          sMessage = Expand('Backup of rule file (%cCubeRuleFileName%) has failed, rule was not loaded.');
          nErrors = nErrors + 1;
          LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
          If( pStrictErrorHandling = 1 ); 
              ProcessQuit; 
          Else;
              ProcessBreak;
          EndIf;
        EndIf;
      Else;
        ##Unloading the Rule###
        ##Before unloading, backup the existing rule (saved as .bkp.txt in data directory or with the suffix parameter)
        ##and drop current rule
        If( sOS @= 'Windows');
          sCmd = 'cmd /c "copy """' | cCubeRuleFileName | '"""  """' | cBackupDirFile |'""" "';
        Else;
          sCmd = 'cp "' | cCubeRuleFileName | '" "' | cBackupDirFile |'"';
        EndIf;
        ExecuteCommand(sCmd,1);
        ## Check if the backup file exists
        If( FileExists( cBackupDirFile ) = 0 );
          sMessage = Expand('Backup of rule file (%cCubeRuleFileName%) has failed, rule was not unloaded.');
          nErrors = nErrors + 1;
          LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
          If( pStrictErrorHandling = 1 ); 
              ProcessQuit; 
          Else;
              ProcessBreak;
          EndIf;
        EndIf;
        If( sOS @= 'Windows');
          sCmd = 'cmd /c "copy """' | cCubeRuleFileName | '"""  """' | cStoreDirFile |'""" "';
        Else;
          sCmd = 'cp "' | cCubeRuleFileName | '"  "' | cStoreDirFile |'"';
        EndIf;
        ExecuteCommand(sCmd,1);
        ## Check if the saved rule file exists
        If( FileExists( cStoreDirFile ) <> 0 );
          CubeRuleDestroy( sCube );
        Else;
          sMessage = Expand('Copy of rule file (%cCubeRuleFileName%) has failed, rule was not unloaded.');
          nErrors = nErrors + 1;
          LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
          If( pStrictErrorHandling = 1 ); 
              ProcessQuit; 
          Else;
              ProcessBreak;
          EndIf;
        EndIf;
      Endif;
      
    Else;
      ###To create a new rule file for the cube (no existing rule)
      If(Upper(pMode) @= 'LOAD');
        RuleLoadFromFile( sCube, cStoreDirFile );
      Else;
        nErrors = nErrors + 1;
        sMessage = 'No Rule file found for cube: ' | sCube;
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
      Endif;
    Endif;
  End;
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

################################################################################################# ####################
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully %pMode% cube rule from cube %pCube% .' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion