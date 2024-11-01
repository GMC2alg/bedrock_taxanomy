#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.dim.attr.importfromfile', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pSrcDir', '', 'pSrcFile', '',
    	'pTitleRows', 1, 'pDelim', ',', 'pQuote', '"'
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
# This process will create Attributes of the dimension from a file.

# Use case: Intended for development/prototyping. 
#1/ Add multiple dimension attributes of different types.

# Note:
# The file format is as per the dimension export file applied to an }ElementsAttributes dimension.
# Naturally, a valid diension name (pDim) is mandatory otherwise the process will abort.
# Also, valid path (pSrcDir) & file name (pSrcFile) are mandatory otherwise the process will abort.
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName     = GetProcessName();
cUserName         = TM1User();
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pSrcDir:%pSrcDir%, pSrcFile:%pSrcFile%, pTitleRows:%pTitleRows%, pDelim:%pDelim%, pQuote:%pQuote%.'; 
cLenASCIICode = 3;

pDelimiter        = TRIM(pDelim);

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

nMetaDataCount = 0;

### Validate Parameters ###

nErrors = 0;

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

# Validate source dir
If(Trim( pSrcDir ) @= '' );
    pSrcDir = GetProcessErrorFileDirectory;
EndIf;
If( SubSt( pSrcDir, Long( pSrcDir ), 1 ) @= sOSDelim );
    pSrcDir = SubSt( pSrcDir, 1, Long( pSrcDir ) -1 );
EndIf;
If( FileExists( pSrcDir ) = 0 );
    nErrors = 1;
    sMessage = 'Invalid source directory specified: folder does not exist.';
    DataSourceType = 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
pSrcDir = pSrcDir | sOSDelim;

# Validate source file
sFile = pSrcDir | pSrcFile;
IF ( Trim ( pSrcFile ) @= '' );
    nErrors = 1;
    sMessage = 'No source file specified.';
    DataSourceType = 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( FileExists( sFile ) = 0 );
    nErrors = 1;
    sMessage = 'Invalid path or source file specified: It does not exist.';
    DataSourceType = 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate dimension
If( Trim( pDim ) @= '' );
    nErrors = 1;
    sMessage = 'No dimension specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( DimensionExists( pDim ) = 0 );
    nErrors = 1;
    sMessage = 'Dimension: ' | pDim | ' does not exist on server.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate file delimiter & quote character
If( pDelimiter @= '' );
    pDelimiter = ',';
Else;
    # If length of pDelimiter is exactly 3 chars and each of them is decimal digit, then the pDelimiter is entered as ASCII code
    nValid = 0;
    If ( LONG(pDelimiter) = cLenASCIICode );
      nChar = 1;
      While ( nChar <= cLenASCIICode );
        If( CODE( pDelimiter, nChar )>=CODE( '0', 1 ) & CODE( pDelimiter, nChar )<=CODE( '9', 1 ) );
          nValid = 1;
        Else;
          nValid = 0;
        EndIf;
        nChar = nChar + 1;
      End;
    EndIf;
    If ( nValid<>0 );
      pDelimiter=CHAR(StringToNumber( pDelimiter ));
    Else;
      pDelimiter = SubSt( Trim( pDelimiter ), 1, 1 );
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

### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### Assign Datasource ###

DataSourceType              = 'CHARACTERDELIMITED';
DatasourceNameForServer     = sFile;
DatasourceNameForClient     = sFile;
DatasourceASCIIHeaderRecords= pTitleRows;
DatasourceASCIIDelimiter    = pDelimiter;
DatasourceASCIIQuoteCharacter= pQuote;

### End Prolog ###
#endregion
#region Metadata

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

## Metatdata Count
nMetaDataCount = nMetaDataCount + 1;

### Write data from source file to target dimension ###
sAttrType = Upper(SubSt( vAttrType, 1, 1 ));

### Validate Record ###
IF(
  sAttrType @= 'A' %
  sAttrType @= 'N' %
  sAttrType @= 'S' );
  ## Continute
ELSE;
  sMessage = 'Invalid attribute type.';
     ITEMSKIP;
ENDIF;

AttrInsert( pDim, '', vAttr, sAttrType );

### End Metadata ###
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully created attributes in %pDim% from file %pSrcFile%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion