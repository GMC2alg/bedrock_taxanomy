#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.dimension.add', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pCube', '', 'pDim', '', 'pDimIndex', 1,
    	'pIncludeData', 0, 'pEle', '', 'pIncludeRules', 2,
    	'pCtrlObj', 0, 'pTemp', 1
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
# This TI adds a dimension to a cube that has already been built with the ability to preserve data.

# Use case: Intended for development/prototyping.
# 1/ Rebuild existing cube with extra dimension without losing data.

# Note:
# Naturally, a valid target cube name (pCube) is mandatory otherwise the process will abort.
# Also, a valid new dimension name (pDim) is mandatory otherwise the process will abort.
# When data needs to be kept (using pIncludeData) a valid element (pEle) must be specified where to store the data in new dimension.
# Rule can be kept as backup file only or reloaded back.
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
cMsgErrorContent= 'Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo        = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pDim:%pDim%, pDimIndex:%pDimIndex%, pIncludeData:%pIncludeData%, pEle:%pEle%, pIncludeRules:%pIncludeRules%, pCtrlObj:%pCtrlObj%, pTemp:%pTemp%.';

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

### Validate Parameters ###
nErrors = 0;
# Validate cube
cDimCount = 0;
If( Trim( pCube ) @= '' );
    nErrors = nErrors + 1;
    sMessage = 'No cube specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( pCube ) = 0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'Invalid cube specified: %pCube%.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Else;
    cDimCount = CubeDimensionCountGet(pCube);
EndIf;

# Don't allow system cubes to be modified
If( SubSt( pCube, 1, 1 ) @= '}' & pCtrlObj <= 0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'Do not modify system cubes: %pCube%.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate index
If( pDimIndex = 0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'Index for new dimension is a REQUIRED parameter!');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( pDimIndex > cDimCount + 1 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'Cube %pCube% has %cDimCount% dimensions. %pDimIndex% is not valid for the new dimension index!');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate dimension
If( Trim( pDim ) @= '' );
    nErrors = nErrors + 1;
    sMessage = 'No dimension specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( DimensionExists( pDim ) = 0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'New dimension: %pDim% does not exist');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# check element chosen in new dimension
If( pIncludeData = 1 & Trim(pEle)@='' );
    nErrors = nErrors + 1;
    sMessage = Expand( 'No element specified in new dimension %pDim% to store cube data.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( pIncludeData = 1 & DIMIX(pDim, pEle)=0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'Invalid element %pEle% specified for the new dimension %pDim% to store cube data.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

IF(pIncludeRules = 1 % pIncludeRules = 2);
    cCubeRuleFileName = '.' | sOSDelim | pCube | '.rux';
    If(FileExists(Lower(cCubeRuleFileName)) = 0);
        pIncludeRules = 0;
        LogOutput( 'INFO', Expand( 'No rule found for %pCube%.' ) );
    Endif;
Endif;  

### Determine number of dims in source cube & create strings to check and recreate ###
nCount = 1;
sDimString = '';
sDimCheck = '';
sDelim = '+';
nNewFound = 0;
nIncrement = 1;
While( TabDim( pCube, nCount ) @<> '' );
  sDim = TabDim( pCube, nCount );
  IF(nCount = pDimIndex & nNewFound = 0);
    sNewDim = pDim;
    nNewFound = 1;
    nIncrement = 0;
  else;
    sNewDim = sDim;
    nIncrement = 1;
  Endif;  
  sDimCheck = sDimCheck|'+'|sDim|'+';
  sDimString = sDimString|'+'|sNewDim;
  nCount = nCount + nIncrement;
End;
nDimensionCount = nCount;

# Cover case of new dimension in last position (pDimIndex = cDimCount+1)
If( nDimensionCount = pDimIndex & nNewFound = 0 );
    nNewFound = 1;
    sNewDim=pDim;
    sDimString = sDimString|'+'|sNewDim;
EndIf;

# Remove any leading +
IF( Subst( sDimString , 1 , 1 ) @= '+' );
    sDimString      = Subst ( sDimString , 2, Long(sDimString)-1 );
EndIf;

IF( Scan('+'|Lower(pDim)|'+', Lower(sDimCheck)) > 0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'The chosen new dimension %pDim% already exists in cube %pCube%.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;

# Check if cube exceeds current max dimenions
If( nDimensionCount > 27 );
    sMessage = 'Process needs to be modified to handle cubes with more than 27 dimensions';
    nErrors = nErrors + 1;
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

######  CALLING THE STEP PROCESSES #####

# Keep the rule
IF(pIncludeRules = 1 % pIncludeRules = 2);
  
  sProc = '}bedrock.cube.rule.manage';
  nRet = ExecuteProcess( sProc,
    'pLogOutput', pLogOutput,
    'pStrictErrorHandling', pStrictErrorHandling,
    'pCube', pCube,
    'pMode', 'UNLOAD'
    );
    
  IF(nRet <> 0);
    sMessage = 'Error unloading the rule for %pCube%.';
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    Else;
        ProcessBreak;
    EndIf;
  ENDIF;
  
Endif; 

# create clone cube with data
IF(pIncludeData = 1);
  
    pCloneCube = pCube | '_Clone';
    nIncludeRules = IF(pIncludeRules = 1 % pIncludeRules = 2, 1, 0);
    nSuppressRules = IF(nIncludeRules = 1,  1, 0);
  
    sProc = '}bedrock.cube.clone';
    nRet = ExecuteProcess( sProc,
        'pLogOutput', pLogOutput,
        'pStrictErrorHandling', pStrictErrorHandling,
        'pSrcCube', pCube,
        'pTgtCube', pCloneCube,
        'pIncludeRules', nIncludeRules,
        'pIncludeData', pIncludeData,
        'pSuppressRules', nSuppressRules,
        'pTemp', pTemp,
        'pCubeLogging', 0
        );

    IF(nRet <> 0);
        sMessage = 'Error creating cloned cube for keeping data.';
        nErrors = nErrors + 1;
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
        If( pStrictErrorHandling = 1 ); 
            ProcessQuit; 
        Else;
            ProcessBreak;
        EndIf;
    ENDIF;
Endif;

# recreate the cube
sProc = '}bedrock.cube.create';
nRet = ExecuteProcess( sProc,
    'pLogOutput', pLogOutput,
    'pStrictErrorHandling', pStrictErrorHandling,
    'pCube', pCube,
    'pDims', sDimString,
    'pRecreate', 1,
    'pDelim', sDelim
    );

IF(nRet <> 0);
    sMessage = Expand('Error recreating the cube: %pCube%.');
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    Else;
        ProcessBreak;
    EndIf;
ENDIF;

# copy back the data
IF(pIncludeData = 1);
    sEleStartDelim = '¦';
    sMappingToNewDims = pDim|sEleStartDelim|pEle;
  
    nRet = ExecuteProcess('}bedrock.cube.data.copy.intercube',
  	'pLogOutput',pLogOutput,
  	'pStrictErrorHandling', pStrictErrorHandling,
  	'pSrcCube',pCloneCube,
  	'pFilter','',
  	'pTgtCube',pCube,
  	'pMappingToNewDims',sMappingToNewDims,
        'pSuppressConsol', 1,
        'pSuppressRules', nSuppressRules,
  	'pZeroTarget',0,
  	'pZeroSource',0,
  	'pFactor',1,
  	'pDimDelim','&',
  	'pEleStartDelim',sEleStartDelim,
  	'pEleDelim','+',
  	'pTemp',pTemp,
  	'pCubeLogging',0);
    
    IF(nRet <> 0);
        sMessage = Expand('Error copying back the data from clone cube: %pCloneCube%.');
        nErrors = nErrors + 1;
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
        If( pStrictErrorHandling = 1 ); 
            ProcessQuit; 
        Else;
            ProcessBreak;
        EndIf;
    ENDIF;
  
    # destroy clone cube
    IF(pTemp=1);
        sProc = '}bedrock.cube.delete';
        nRet = ExecuteProcess( sProc,
            'pLogOutput', pLogOutput,
            'pStrictErrorHandling', pStrictErrorHandling,
            'pCube', pCloneCube,
            'pCtrlObj', 0
            );

        IF(nRet <> 0);
            sMessage = Expand('Error deleting the clone cube: %pCloneCube%.');
            nErrors = nErrors + 1;
            LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
            If( pStrictErrorHandling = 1 ); 
                ProcessQuit; 
            Else;
                ProcessBreak;
            EndIf;
        ENDIF;
    Endif;

Endif; 

# reload the rule
IF(pIncludeRules = 2);
  
    sProc = '}bedrock.cube.rule.manage';

    nRet = ExecuteProcess( sProc,
        'pLogOutput', pLogOutput,
        'pStrictErrorHandling', pStrictErrorHandling,
        'pCube', pCube,
        'pMode', 'LOAD'
        );
    
    IF(nRet <> 0);
      sMessage = Expand('Error reloading the rule for %pCube%.');
      nErrors = nErrors + 1;
      LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
      # Create error rule file 
      cErrorRuleName = 'ErrorRuleFile.rux';
      
      IF(FileExists( Lower(cErrorRuleName) ) = 0 );
        sFile = '.' | sOSDelim | Lower(cErrorRuleName);
        LogOutput(cMsgErrorLevel, 'Rule could not be attached due to invalid !Dimension references. Please recover from the backup and fix manually.');
      ENDIF;
      
      ExecuteProcess( sProc,
      'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
      'pCube', pCube,
      'pFileName', Lower(cErrorRuleName),
      'pMode', 'LOAD'
      );
      If( pStrictErrorHandling = 1 ); 
          ProcessQuit; 
      Else;
          ProcessBreak;
      EndIf;
    ENDIF;
  
Endif; 

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully added dimension %pDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion