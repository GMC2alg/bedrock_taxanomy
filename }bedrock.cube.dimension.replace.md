#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.dimension.replace', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pCube', '', 'pSrcDim', '', 'pTgtDim', '',
    	'pIncludeData', 0, 'pEle', '',
    	'pIncludeRules', 0,
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
# This TI deletes a dimension and adds another one to an existing cube with the ability to preserve data.

# Use case: Intended for development/prototyping.
# 1/ Rebuild existing cube after removal of one dimension and adding anothr one without losing all the data.

# Note:
# Naturally, a valid cube name (pCube) is mandatory otherwise the process will abort.
# Also, valid dimension names (pSrcDim & pTgtDim) are mandatory otherwise the process will abort.
# When data needs to be kept (using pIncludeData) a valid element (pEle) in new dimension must be specified
# where to store the data. Data is summed from original dimension.
# Rule can be kept as backup file only or reloaded back.
#EndRegion @DOC


# This process selects the cube and replaces
# the source dimensions from a specified target dimensions.
# This process should only be run on an EMPTY cube or a cube that
# has already had all data exported to a text file

# Note:
# - This process does not preserve any cube rules as they would likely no longer be
#   valid if a dimension is replaced

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');

### Constants ###
cThisProcName     = GetProcessName();
cUserName         = TM1User();
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pSrcDim:%pSrcDim%, pTgtDim:%pTgtDim%, pIncludeData:%pIncludeData%, pEle:%pEle%, pIncludeRules:%pIncludeRules%, pCtrlObj:%pCtrlObj%, pTemp:%pTemp%.' ;   
cDefaultView      = Expand( '%cThisProcName%_%cTimeStamp%_%cRandomInt%' );

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

### Validate Parameters ###
nErrors = 0;
# Validate cube
If( Trim( pCube ) @= '' );
    sMessage = 'No cube specified.';
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( pCube ) = 0 );
    sMessage = Expand( 'Cube %pCube% does not exist.' );
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Don't allow system cubes to be modified
If( SubSt( pCube, 1, 1 ) @= '}' & pCtrlObj <= 0 );
  nErrors = nErrors + 1;
  sMessage = Expand( 'Do not modify system cubes: %pCube%.');
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate source dimension
If( Trim( pSrcDim ) @= '' );
  nErrors = nErrors + 1;
  sMessage = 'No source dimension specified.';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( DimensionExists( pSrcDim ) = 0 );
  nErrors = nErrors + 1;
  sMessage = Expand( 'Source dimension %pSrcDim% does not exist.');
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate target dimension
If( Trim( pTgtDim ) @= '' );
  nErrors = nErrors + 1;
  sMessage = 'No target dimension specified.';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( DimensionExists( pTgtDim ) = 0 );
  nErrors = nErrors + 1;
  sMessage = Expand( 'Target dimension %pTgtDim% does not exist.');
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Check that the source and target dimensions are different
If( pSrcDim @= pTgtDim );
  sMessage = Expand('Source and target dimensions are the same: %pSrcDim%');
  nErrors = nErrors + 1;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# check element chosen in new dimension
If( pIncludeData = 1 & Trim(pEle)@='' );
  nErrors = nErrors + 1;
  sMessage = Expand( 'No element specified in new dimension %pTgtDim% to store cube data.');
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( pIncludeData = 1 & DIMIX(pTgtDim, pEle)=0 );
  nErrors = nErrors + 1;
  sMessage = Expand( 'Invalid element %pEle% specified for the new dimension %pTgtDim% to store cube data.');
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

IF(pIncludeRules = 1 % pIncludeRules = 2);
  cCubeRuleFileName = '.' | sOSDelim | pCube | '.RUX';
  If(FileExists(cCubeRuleFileName) = 0);
    pIncludeRules = 0;
    LogOutput( 'INFO', Expand( 'No rule found for %pCube%.' ) );
  Endif;
Endif;  

### Determine number of dims in source cube & create strings to check and recreate ###
nCount = 1;
sDimString = '';
sDimCheck = '';
sDelim = '+';
While( TabDim( pCube, nCount ) @<> '' );
  sDim = TabDim( pCube, nCount );
  IF(sDim@=pSrcDim);
    sNewDim=pTgtDim;
  else;
    sNewDim=sDim;
  Endif;  
  IF(nCount = 1);
    sDimCheck = '+'|sDim|'+';
    sDimString = sNewDim;
  elseif(nCount > 1);
    sDimCheck = sDimCheck|'+'|sDim|'+';
    sDimString = sDimString|'+'|sNewDim;
  Endif;
  nCount = nCount + 1;
End;
nDimensionCount = nCount - 1;

#Remove any leading +
IF( Subst( sDimString , 1 , 1 ) @= '+' );
    sDimString      = Subst ( sDimString , 2 , 999 );
EndIf;

#check source dimension
IF(scan('+'|pSrcDim|'+', sDimCheck)=0);
  nErrors = nErrors + 1;
  sMessage = Expand( 'Source Dimension %pSrcDim% does not exist in %pCube%.');
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
Endif;  

#check target dimension
IF(scan('+'|pTgtDim|'+', sDimCheck)>0);
  nErrors = nErrors + 1;
  sMessage = Expand( 'Target Dimension %pTgtDim% already exists in %pCube%.');
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
    sMappingToNewDims = pTgtDim|sEleStartDelim|pEle;
  
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

      IF(FileExists( cErrorRuleName ) = 0 );
        sFile = '.' | sOSDelim | cErrorRuleName;
        LogOutput(cMsgErrorLevel, 'Rule could not be attached due to invalid !Dimension references. Please recover from the backup and fix manually.');
      ENDIF;

      ExecuteProcess( sProc,
      'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
      'pCube', pCube,
      'pFileName', cErrorRuleName,
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully replaced the %pSrcDim% dimension with the %pTgtDim% in the %pCube% cube. Data was loaded to the %pEle% item.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;
  
### End Epilog ###
#endregion