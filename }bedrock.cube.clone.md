#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.clone', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pSrcCube' ,'', 'pTgtCube', '',
    	'pIncludeRules', 1, 'pIncludeData', 0,
    	'pFilter', '',
    	'pDimDelim', '&', 'pEleStartDelim', '¦', 'pEleDelim', '+',
    	'pSuppressRules', 1, 'pTemp', 1, 'pCubeLogging', 0
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
# This process replicates an existing cube. It can include data & rules too.

# Use case: Intended for development/prototyping.
# 1. Take a snapshot of cube data copying all rules to values.
# 2. Take an exact copy of a cube in a "one click action" as a starting point for prototyping rule changes or developing new features.

# Note:
# * There are parameter options to include data (pIncludeData) and rules (pIncludeRules) with the creation of the cube.
# * If the source cube (pSrcCube) is left blank or doesn't exist in the model, process will terminate withoud doing anything.
# * If the target cube (pTgtCube) already exists in the model, process will terminate withoud doing anything.
# * If the target cube is left blank or is the same as the source cube the cloned cube will inherit the source cube name with "_Clone" appended.
# * If the source cube data only needs to be partially copied, then the pFilter parameter should be entered otherwise all other parameters can be left as is.
# * In productive systems this process may be called internally by other processes (}bedrock.cube.data.copy, }bedrock.cube.data.copy.intercube) if copying data via intermediate cloned cube.
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName   = GetProcessName();
cUserName       = TM1User();
cTimeStamp      = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt      = NumberToString( INT( RAND( ) * 1000 ));
cTempSub        = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel  = 'ERROR';
cMsgErrorContent= 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo        = 'Process:%cThisProcName% run with parameters pSrcCube:%pSrcCube%, pTgtCube:%pTgtCube%, pIncludeRules:%pIncludeRules%, pIncludeData:%pIncludeData%, pFilter:%pFilter%, pDimDelim:%pDimDelim%, pEleStartDelim:%pEleStartDelim%, pEleDelim:%pEleDelim%, pSuppressRules:%pSuppressRules%, pTemp:%pTemp%, pCubeLogging:%pCubeLogging%.' ;   
cDimCountMax    = 30 ;

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Initialise ###
nErrors         = 0;
nDataCheck      = 0;
sDimCountMax    = NumberToString( cDimCountMax );
sDimsString     = '';
sDelim          = '+';

### Validate Parameters ###

## Default filter delimiters
If( pDimDelim     @= '' );
    pDimDelim     = '&';
EndIf;
If( pEleStartDelim@= '' );
    pEleStartDelim= '¦';
EndIf;
If( pEleDelim     @= '' );
    pEleDelim     = '+';
EndIf;

# Validate source cube
If( Trim( pSrcCube ) @= '' );
    nErrors = 1;
    sMessage = 'No cube specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( pSrcCube ) = 0 );    
    sMessage = Expand( 'Invalid source cube specified: %pSrcCube%.' );
    nErrors = 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate target cube
If( pTgtCube @= '' % pTgtCube @= pSrcCube );
    pTgtCube = pSrcCube | '_Clone';
EndIf;
If( CubeExists( pTgtCube ) = 1 );    
    sMessage = Expand( 'Invalid target cube : %pTgtCube%.' );
    nErrors = 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

### Create the clone cube ###
nDimCount = 1;
While( TabDim( pSrcCube, nDimCount ) @<> '' );
  sDimName = TabDim (pSrcCube, nDimCount);
  sDimsString = sDimsString | sDimName | sDelim;
  nDimCount = nDimCount + 1;
End;
nDimCount = nDimCount - 1;
sDimsString = Subst(sDimsString,1,long(sDimsString)-long(sDelim));

If( nDimCount > cDimCountMax );
  nErrors = 1;
  sMessage = Expand( 'Cube has too many dimensions: %pSrcCube% max %sDimCountMax% dims catered for, TI must be altered to accomodate.' );
  DataSourceType = 'NULL';
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

sProc = '}bedrock.cube.create';
nRet = ExecuteProcess( sProc,
  'pLogOutput', pLogOutput,
  'pStrictErrorHandling', pStrictErrorHandling,
  'pCube', pTgtCube,
  'pDims', sDimsString,
  'pRecreate', 1,
  'pDelim', sDelim
  );

IF(nRet <> 0);
  sMessage = 'Error creating the target cube.';
  nErrors = 1;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
ENDIF;


### copy data ####
If( pIncludeData = 1 );
nRet = ExecuteProcess('}bedrock.cube.data.copy.intercube',
    'pLogOutput', pLogOutput,
    'pStrictErrorHandling', pStrictErrorHandling,
  	'pSrcCube',pSrcCube,
  	'pFilter',pFilter,
  	'pTgtCube',pTgtCube,
  	'pMappingToNewDims','',
  	'pSuppressConsol',1,
  	'pSuppressRules',pSuppressRules,
  	'pZeroTarget',0,
  	'pZeroSource',0,
  	'pFactor',1,
    'pDimDelim', pDimDelim,
    'pEleStartDelim', pEleStartDelim,
    'pEleDelim', pEleDelim,
    'pTemp', pTemp,
    'pCubeLogging', pCubeLogging);

  IF(nRet <> 0);
    sMessage = 'Error copying data.';
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    Else;
        ProcessBreak;
    EndIf;
  ENDIF;

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

### Attach rules to cloned cube ###
If( nErrors = 0 & pIncludeRules = 1 );
  sRuleFile = pSrcCube | '.rux';
  If( FileExists( sRuleFile ) = 1 );
    If( nErrors = 0 );
      RuleLoadFromFile( pTgtCube, sRuleFile );
    EndIf;
  EndIf;
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully cloned the %pSrcCube% cube to %pTgtCube%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion