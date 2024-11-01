#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.data.export', 'pLogoutput', pLogoutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pCube', '', 'pView', '', 'pFilter', '',
    	'pFilterParallel', '', 'pParallelThreads', 0,
    	'pDimDelim', '&', 'pEleStartDelim', '¦', 'pEleDelim', '+',
    	'pSuppressZero', 1, 'pSuppressConsol', 1, 'pSuppressRules', 1, 'pSuppressConsolStrings', 1,
    	'pZeroSource', 0, 'pCubeLogging', 0, 'pTemp', 1,
    	'pFilePath', '', 'pFileName', '',
    	'pDelim', ',','pDecimalSeparator','.','pThousandSeparator',',',
      'pQuote', '"', 'pTitleRecord', 1, 'pSandbox', pSandbox, 'pSubN', pSubN, 'pCubeNameExport', pCubeNameExport
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
# This TI is designed to export data in a given cube to a flat file for a given "slice" (any dimension/element combination).
#
# Use case: Intended for development/prototyping or in Production environment.
# 1. Export data for import into another TM1 model to eliminate possibility of locking.
# 2. Export data for import into ERP system.
#
# Note:
# * Naturally, a valid cube name (pCube) is mandatory otherwise the process will abort.
# * All other parameters are optional, however, the filter (pFilter) should be specified to limit the size of the file.
# * The default output path is the same as the error file path.
# * As this TI has a view as a data source it requires the implicit variables NValue, SValue and Value_is_String
# * To edit this TI in Architect a tmp cube with minimum 24 dims is needed as the preview data source or set the data
#   source to ASCII and manually edit the TI in notepad after saving to add back the required implicit view variables
# * If using the pFilterParallel parameter the **single dimension** used as the "parallelization slicer" cannot appear in
#   the pFilter parameter
# * When using parallelization via the *RunProcess* function the elements listed in pFilterParallel will be split one_at_a_time
#   and passed to a recursive call of the process being added to pFilter. Each element name will also be appended to the filename
#
# Warning:
# As the *RunProcess* function currently has no mechanism to check for the state of the called process if more processes are
# released than available CPU cores on the server then this could lead to TM1 consuming all available server resources and a
# associated performance issue. Be careful that the number of slicer elements listed in pFilterParallel should not exceed the
# number of available cores.
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
StringGlobalVariable('sBedrockViewCreateParsedFilter');

### Constants ###
cThisProcName     = GetProcessName();
cUserName         = TM1User();
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pView:%pView%, pFilter:%pFilter%, pFilterParallel:%pFilterParallel%, pParallelThreads:%pParallelThreads%, pDimDelim:%pDimDelim%, pEleStartDelim:%pEleStartDelim%, pEleDelim:%pEleDelim%, pSuppressZero:%pSuppressZero%, pSuppressConsol:%pSuppressConsol%, pSuppressRules:%pSuppressRules%, pZeroSource:%pZeroSource%, pCubeLogging:%pCubeLogging%, pTemp:%pTemp%, pFilePath:%pFilePath%, pFileName:%pFileName%, pDelim:%pDelim%, pQuote:%pQuote%, pTitleRecord:%pTitleRecord%, pSandbox:%pSandbox%, pSuppressConsolStrings:%pSuppressConsolStrings%.';
cDefaultView      = Expand( '%cThisProcName%_%cTimeStamp%_%cRandomInt%' );
cLenASCIICode     = 3;

pFieldDelim       = TRIM(pDelim);
pDimDelim         = TRIM(pDimDelim);
pEleStartDelim    = TRIM(pEleStartDelim);
pEleDelim         = TRIM(pEleDelim);
pDecimalSeparator = TRIM(pDecimalSeparator);
pThousandSeparator= TRIM(pThousandSeparator);
nDataCount        = 0;
nErrors           = 0;

## Capture current transaction logging state
If( CubeExists(pCube) = 1 );
  sCubeLogging = CellGetS('}CubeProperties', pCube, 'LOGGING' );
EndIf;

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
If( pDecimalSeparator @= '' );
 	pDecimalSeparator = '.';
EndIf;
If( pThousandSeparator @= '' );
 	pThousandSeparator = ',';
EndIf;
sDelimDim = pDimDelim;
sElementStartDelim = pEleStartDelim;
sDelimelem = pEleDelim;

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );
ENDIF;

### Validate Parameters ###

# If no cube has been specified then terminate process
If( Trim( pCube ) @= '' );
    sMessage = 'No cube specified.';
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( pCube ) = 0 );
    sMessage = Expand( 'Cube: %pCube% does not exist.' );
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate the View parameter
If( TRIM(pView) @= '' );
    cView = cDefaultView ;
Else ;
    cView = pView ;
EndIf;
cSubset = cView;

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

# Validate file path
If(Trim( pFilePath ) @= '' );
    pFilePath = GetProcessErrorFileDirectory;
EndIf;
If( SubSt( pFilePath, Long( pFilePath ), 1 ) @= sOSDelim );
    pFilePath = SubSt( pFilePath, 1, Long( pFilePath ) -1 );
EndIf;
If(  FileExists( pFilePath ) = 0 );
    sMessage = Expand('Invalid export directory: %pFilePath%');
    nErrors = nErrors + 1;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
pFilePath = pFilePath | sOSDelim;

# Validate file name
If( pFileName @= '' );
    sBaseFileName = Expand('%pCube%_Export');
    sExt = '.csv';
    pFileName = sBaseFileName | '.csv';
Else;
    # determine file extension. If no file extension entered then use .csv as default
    If( Scan( '.', pFileName ) = 0 );
        sExt = '.csv';
        sBaseFileName = pFileName;
    Else;
        sExt = SubSt( pFileName, Scan( '.', pFileName ), Long( pFileName ) );
        sBaseFileName = SubSt( pFileName, 1, Scan( '.', pFileName ) - 1 );
    EndIf;
    pFileName = sBaseFileName | sExt;
EndIf;
cExportFile = pFilePath | pFileName;

# Validate parallelization filter
If( Scan( pEleStartDelim, pFilterParallel ) > 0 );
    sDimParallel = SubSt( pFilterParallel, 1, Scan( pEleStartDelim, pFilterParallel ) - 1 );
    If( Scan( Lower(sDimParallel) | pEleStartDelim, Lower(pFilter) ) > 0 );
        sMessage = 'Parallelization dimension %sDimParallel% cannot exist in filter.';
        nErrors = nErrors + 1;
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    EndIf;
EndIf;

# Validate Max Threads
If( pParallelThreads > 0 );
  nMaxThreads = pParallelThreads;
Else;
  nMaxThreads = 1;
EndIf;

# Validate file delimiter & quote character
If( pFieldDelim @= '' );
    pFieldDelim = ',';
Else;
    # If length of pFieldDelim is exactly 3 chars and each of them is decimal digit, then the pFieldDelim is entered as ASCII code
    nValid = 0;
    If ( LONG(pFieldDelim) = cLenASCIICode );
      nChar = 1;
      While ( nChar <= cLenASCIICode );
        If( CODE( pFieldDelim, nChar ) >= CODE( '0', 1 ) & CODE( pFieldDelim, nChar ) <= CODE( '9', 1 ) );
          nValid = 1;
        Else;
          nValid = 0;
          Break;
        EndIf;
        nChar = nChar + 1;
      End;
    EndIf;
    If ( nValid<>0 );
      pFieldDelim=CHAR(StringToNumber( pFieldDelim ));
    Else;
      pFieldDelim = SubSt( Trim( pFieldDelim ), 1, 1 );
    EndIf;
EndIf;

If( pQuote @= '' );
    ## Use no quote character
Else;
    # If length of pQuote is exactly 3 chars and each of them is decimal digit, then the pQuote is entered as ASCII code
    nValid = 0;
    If ( LONG(pQuote) = cLenASCIICode );
      nChar = 1;
      While ( nChar <= cLenASCIICode );
        If( CODE( pQuote, nChar ) >= CODE( '0', 1 ) & CODE( pQuote, nChar ) <= CODE( '9', 1 ) );
          nValid = 1;
        Else;
          nValid = 0;
          Break;
        EndIf;
        nChar = nChar + 1;
      End;
    EndIf;
    If ( nValid<>0 );
      pQuote=CHAR(StringToNumber( pQuote ));
    Else;
      pQuote = SubSt( Trim( pQuote ), 1, 1 );
    EndIf;
EndIf;

If ( LONG(pDecimalSeparator) = cLenASCIICode );
  nValid = 0;
  nChar = 1;
  While ( nChar <= cLenASCIICode );
    If( CODE( pDecimalSeparator, nChar ) >= CODE( '0', 1 ) & CODE( pDecimalSeparator, nChar ) <= CODE( '9', 1 ) );
      nValid = 1;
    Else;
      nValid = 0;
      Break;
    EndIf;
    nChar = nChar + 1;
  End;
  If ( nValid<>0 );
    pDecimalSeparator = CHAR(StringToNumber( pDecimalSeparator ));
  Else;
    pDecimalSeparator = SubSt( Trim( pDecimalSeparator ), 1, 1 );
  EndIf;
EndIf;
sDecimalSeparator = pDecimalSeparator;

If ( LONG(pThousandSeparator) = cLenASCIICode );
  nValid = 0;
  nChar = 1;
  While ( nChar <= cLenASCIICode );
    If( CODE( pThousandSeparator, nChar ) >= CODE( '0', 1 ) & CODE( pThousandSeparator, nChar ) <= CODE( '9', 1 ) );
      nValid = 1;
    Else;
      nValid = 0;
      Break;
    EndIf;
    nChar = nChar + 1;
  End;
  If ( nValid<>0 );
    pThousandSeparator = CHAR(StringToNumber( pThousandSeparator ));
  Else;
    pThousandSeparator = SubSt( Trim( pThousandSeparator ), 1, 1 );
  EndIf;
EndIf;
sThousandSeparator = pThousandSeparator;

# Validate Sandbox
If( TRIM( pSandbox ) @<> '' );
    If( ServerSandboxExists( pSandbox ) = 0 );
        SetUseActiveSandboxProperty( 0 );
        nErrors = nErrors + 1;
        sMessage = Expand('Sandbox %pSandbox% is invalid for the current user.');
        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    Else;
        ServerActiveSandboxSet( pSandbox );
        SetUseActiveSandboxProperty( 1 );
    EndIf;
Else;
    SetUseActiveSandboxProperty( 0 );
EndIf;

# Validate Character Set
If(Trim( pCharacterSet ) @= '' );
  pCharacterSet = 'TM1CS_UTF8';
EndIf;

# Jump to Epilog if any errors so far
IF ( nErrors > 0 );
    DataSourceType = 'NULL';
    If( pStrictErrorHandling = 1 );
        If( CubeExists(pCube) = 1 );
          CubeSetLogChanges( pCube, IF(sCubeLogging@='YES',1,0) );
        EndIf;
        ProcessQuit;
    Else;
        ProcessBreak;
    EndIf;
ENDIF;

# Branch depending on whether to do recursive calls to self on independent threads or run all in this thread
If( Scan( pEleStartDelim, pFilterParallel ) > 0 );
  sDimParallel = SubSt( pFilterParallel, 1, Scan( pEleStartDelim, pFilterParallel ) - 1 );
  sElementList = SubSt( pFilterParallel, Scan( pEleStartDelim, pFilterParallel ) + 1, Long( pFilterParallel ) );
  If( SubSt( sElementList, Long( sElementList ), 1 ) @<> pEleDelim );
      sElementList = sElementList | pEleDelim;
  EndIf;
  ## Counting elements in element list
  sElementListCount = sElementList;
  nElements = 0;
  While( Scan( pEleDelim, sElementListCount ) > 0 );
    nElements = nElements + 1;
    sElementListCount = SubSt( sElementListCount, Scan( pEleDelim, sElementListCount ) + 1, Long( sElementListCount ) );
  End;
  IF( Mod( nElements, nMaxThreads ) = 0 );
    nElemsPerThread = INT( nElements \ nMaxThreads );
  ELSE;
    nElemsPerThread = INT( nElements \ nMaxThreads ) + 1;
  ENDIF;
  nThreadElCounter = 0;
  While( Scan( pEleDelim, sElementList ) > 0 );
      sSlicerEle = SubSt( sElementList, 1, Scan( pEleDelim, sElementList ) - 1 );
      sElementList = SubSt( sElementList, Scan( pEleDelim, sElementList ) + 1, Long( sElementList ) );
      # Do recursive process call with new RunProcess function
      nThreadElCounter = nThreadElCounter + 1;
      sDimDelim = If(pFilter @= '', '', pDimDelim );
      IF( nThreadElCounter = 1 );
        sFilter = Expand('%pFilter%%sDimDelim%%sDimParallel%%pEleStartDelim%%sSlicerEle%');
        sFileName = Expand('%sBaseFileName%_%sDimParallel%_%sSlicerEle%');
      ELSE;
        sFilter = Expand('%sFilter%%pEleDelim%%sSlicerEle%');
        sFileName = Expand('%sFileName%+%sSlicerEle%');
      ENDIF;
      IF( nThreadElCounter >= nElemsPerThread );
        sFileName = Expand('%sFileName%%sExt%');
        RunProcess( cThisProcName, 'pLogoutput', pLogoutput,
        	'pCube', pCube, 'pView', '',
        	'pFilter', sFilter, 'pFilterParallel', '',
        	'pDimDelim', pDimDelim, 'pEleStartDelim', pEleStartDelim, 'pEleDelim', pEleDelim,
        	'pSuppressZero', pSuppressZero, 'pSuppressConsol', pSuppressConsol, 'pSuppressRules', pSuppressRules,
        	'pZeroSource', pZeroSource, 'pCubeLogging', pCubeLogging,
        	'pTemp', pTemp, 'pFilePath', pFilePath, 'pFileName', sFileName,
        	'pDelim', pFieldDelim, 'pDecimalSeparator', pDecimalSeparator, 'pThousandSeparator', pThousandSeparator,
          'pQuote', pQuote, 'pTitleRecord', pTitleRecord, 'pSandbox', pSandbox, 'pSuppressConsolStrings', pSuppressConsolStrings, 'pCubeNameExport', pCubeNameExport
        );
    	  nThreadElCounter = 0;
    	  sFilter = '';
    	  sFileName = '';
    	 ENDIF;
  End;
  ## Process last elements - only when filter is not empty (there are still elements)
  IF( sFilter @<> '' );
    sFileName = Expand('%sFileName%%sExt%');
    RunProcess( cThisProcName, 'pLogoutput', pLogoutput,
    	'pCube', pCube, 'pView', '',
    	'pFilter', sFilter, 'pFilterParallel', '',
    	'pDimDelim', pDimDelim, 'pEleStartDelim', pEleStartDelim, 'pEleDelim', pEleDelim,
    	'pSuppressZero', pSuppressZero, 'pSuppressConsol', pSuppressConsol, 'pSuppressRules', pSuppressRules,
    	'pZeroSource', pZeroSource, 'pCubeLogging', pCubeLogging,
    	'pTemp', pTemp, 'pFilePath', pFilePath, 'pFileName', sFileName,
    	'pDelim', pFieldDelim, 'pDecimalSeparator', pDecimalSeparator, 'pThousandSeparator', pThousandSeparator,
      'pQuote', pQuote, 'pTitleRecord', pTitleRecord, 'pSandbox', pSandbox, 'pSuppressConsolStrings', pSuppressConsolStrings, 'pCubeNameExport', pCubeNameExport
    );
  ENDIF;
  DataSourceType = 'NULL';
  nParallelRun = 1;
Else;
  # No parallelization is being used. Proceed as normal and do everything internally

  # Determine number of dims in source cube & create strings to expand on title and rows
  nCount = 1;
  nDimensionIndex = 0;

  ## Skip cube name from export
  IF (pCubeNameExport = 0);
    sTitle = '';
    sRow = '';

    While( TabDim( pCube, nCount ) @<> '' );
        sDimension = TabDim( pCube, nCount );

        ## Determine title string for the source cube
        sTitle = sTitle|'%pQuote%'|sDimension|'%pQuote%%pFieldDelim%';
        # Determine row string for the source cube
        sRow = sRow|'%pQuote%%V'| numbertostring(nCount) |'%%pQuote%%pFieldDelim%';

        nCount = nCount + 1;
    End;
    nDimensionCount = nCount - 1;

    # Finish off the strings
    sTitle = sTitle|'%pQuote%Value%pQuote%';
    sRow = sRow|'%pQuote%%sValue%%pQuote%';

  ELSE;
    sTitle = '%pQuote%Cube%pQuote%';
    sRow = '%pQuote%%pCube%%pQuote%';

    While( TabDim( pCube, nCount ) @<> '' );
        sDimension = TabDim( pCube, nCount );

        ## Determine title string for the source cube
        sTitle = sTitle|'%pFieldDelim%%pQuote%'|sDimension|'%pQuote%';
        # Determine row string for the source cube
        sRow = sRow|'%pFieldDelim%%pQuote%%V'| numbertostring(nCount) |'%%pQuote%';

        nCount = nCount + 1;
    End;
    nDimensionCount = nCount - 1;

    # Finish off the strings
    sTitle = sTitle|'%pFieldDelim%%pQuote%Value%pQuote%';
    sRow = sRow|'%pFieldDelim%%pQuote%%sValue%%pQuote%';
  ENDIF;

  # Create Processing View for source version
  nRet = ExecuteProcess('}bedrock.cube.view.create',
          'pLogOutput', pLogOutput,
          'pStrictErrorHandling', pStrictErrorHandling,
          'pCube', pCube,
          'pView', cView,
          'pFilter', pFilter,
          'pSuppressZero', pSuppressZero,
          'pSuppressConsol', pSuppressConsol,
          'pSuppressRules', pSuppressRules,
          'pSuppressConsolStrings', pSuppressConsolStrings,
          'pDimDelim', pDimDelim,
          'pEleStartDelim', pEleStartDelim,
          'pEleDelim', pEleDelim,
          'pTemp', pTemp,
          'pSubN', pSubN
          );

    # Validate Sandbox
    If( TRIM( pSandbox ) @<> '' );
      If( ServerSandboxExists( pSandbox ) = 0 );
          SetUseActiveSandboxProperty( 0 );
          nErrors = nErrors + 1;
          sMessage = Expand('Sandbox %pSandbox% is invalid for the current user.');
          LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
      Else;
          ServerActiveSandboxSet( pSandbox );
          SetUseActiveSandboxProperty( 1 );
      EndIf;
    Else;
      SetUseActiveSandboxProperty( 0 );
    EndIf;


  IF( nRet <> ProcessExitNormal() );
      sMessage = 'Error creating the view from the filter.';
      nErrors = nErrors + 1;
      LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
      If( pStrictErrorHandling = 1 );
          If( CubeExists(pCube) = 1 );
            CubeSetLogChanges( pCube, IF(sCubeLogging@='YES',1,0) );
          EndIf;
          ProcessQuit;
      Else;
          ProcessBreak;
      EndIf;
  ENDIF;

  sParsedFilter = sBedrockViewCreateParsedFilter;
  sFilterRow = '%pQuote%%pCube%%pQuote%%pFieldDelim%%pQuote%Filter%pQuote%%pFieldDelim%%pQuote%%sParsedFilter%%pQuote%%pFieldDelim%%pQuote%%pDimDelim%%pQuote%%pFieldDelim%%pQuote%%pEleStartDelim%%pQuote%%pFieldDelim%%pQuote%%pEleDelim%%pQuote%';

  # Assign Datasource
  DataSourceType          = 'VIEW';
  DatasourceNameForServer = pCube;
  DatasourceNameForClient = pCube;
  DatasourceCubeView      = cView;
  DatasourceAsciiDelimiter= pFieldDelim;
  DatasourceAsciiQuoteCharacter = '';
  nParallelRun = 0;
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

#################################################################################################
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
#################################################################################################

# Set the output character set
SetOutputCharacterSet( cExportFile, pCharacterSet );

### Data Count ###
nDataCount = nDataCount + 1;

# Output the title string
IF( nDataCount = 1 & pTitleRecord >= 1 );
    TextOutput( cExportFile, Expand(sTitle) );
Endif;

### Export filter into the 1st record of the file, it will be used from import process to zero out the corresponding slice, if specified
IF( nDataCount = 1 & pTitleRecord = 2 );
    TextOutput( cExportFile, Expand(sFilterRow) );
Endif;

### Export data from source version to file ###
If( value_is_string = 0 );
    sValue = NumberToStringEx( nValue, '#,0.#############', sDecimalSeparator, sThousandSeparator );
EndIf;

# Selects the correct TextOutput formula depending upon the number of dimensions in the cube
IF(SCAN( CHAR( 10 ), sValue ) > 0 );
    sValueCleaned = '';
    nNoChar = 1;
    nLimit = LONG( sValue );
    WHILE( nNoChar <= nLimit ) ;
        sChar = SUBST(  sValue, nNoChar, 1 );
        IF( CODE( sChar, 1 ) <> 10 );
            sValueCleaned = sValueCleaned | sChar ;
        ELSE;
            sValueCleaned = sValueCleaned | ' ';
        ENDIF;
        nNoChar = nNoChar + 1;
    END;
    sValue = sValueCleaned;
ENDIF;

# Output data
TextOutput( cExportFile, Expand(sRow) );

### End Data ###
#endregion
#region Epilog
#****Begin: Generated Statements***
#****End: Generated Statements****

#################################################################################################
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
#################################################################################################

### Delete source data ###
If( pZeroSource = 1 & nErrors = 0 & nParallelRun = 0 );
    If ( pCubeLogging <= 1 );
      CubeSetLogChanges( pCube, pCubeLogging);
    EndIf;
    ViewZeroOut( pCube, cView );
    If ( pCubeLogging <= 1 );
      CubeSetLogChanges( pCube, IF(sCubeLogging@='YES',1,0) );
    EndIf;
EndIf;

### Return code & final error message handling
If( nErrors > 0 );
    sMessage = 'the process incurred at least 1 error. Please see above lines in this file for more details.';
    nProcessReturnCode = 0;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    sProcessReturnCode = Expand( '%sProcessReturnCode% Process:%cThisProcName% completed with errors. Check tm1server.log for details.' );
    If( pStrictErrorHandling = 1 );
        If( CubeExists(pCube) = 1 );
          CubeSetLogChanges( pCube, IF(sCubeLogging@='YES',1,0) );
        EndIf;
        ProcessQuit;
    EndIf;
Else;
    sDataCount = NUMBERTOSTRING (nDataCount);
    sProcessAction = Expand( 'Process:%cThisProcName% exported %sDataCount% records from %pCube% based on filter %pFilter%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );
    EndIf;

EndIf ;

### End Epilog ###
#endregion