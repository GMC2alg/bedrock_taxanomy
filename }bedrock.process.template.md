#region Prolog
################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

#****Begin: Generated Statements***
#****End: Generated Statements****

################################################################################################# 
### CHANGE HISTORY:
### MODIFICATION DATE 	CHANGED BY 	    COMMENT
### YYYY-MM-DD 		    Developer Name 	Creation of Process
### YYYY-MM-DD 		    Developer Name 	Reason for modification here
################################################################################################# 
#Region @DOC
# Description:
# A description of what this process does here.

# Use case:
# A description of the use cast for this process does here.

# Note:
# * List any notes for users to be aware of here.
#EndRegion @DOC
################################################################################################# 

################################################################################################# 
#Region Process Declarations
### Process Parameters
# a short description of what the process does goes here in cAction variable, e.g. "copied data from cube A to cube B". This will be written to the message log if pLogOutput=1
cAction             = 'ran with no action';
cParamArray         = '';
# to use the parameter array remove the line above and uncomment the line below, adding the needed parameters in the provided format
#cParamArray         = 'pLogOutput:%pLogOutput%, pTemp:%pTemp%';

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');

### Standard Constants
cThisProcName       = GetProcessName();
cUserName           = TM1User();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cTempObjName        = Expand('%cThisProcName%_%cTimeStamp%_%cRandomInt%');
cViewClr            = '}bedrock_clear_' | cTempObjName;
cViewSrc            = '}bedrock_source_' | cTempObjName;
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters %cParamArray%';
sDelimEleStart      = '¦';
sDelimDim           = '&';
sDelimEle           = '+';
nProcessReturnCode  = 0;
nErrors             = 0;
nMetadataRecordCount= 0;
nDataRecordCount    = 0;

### Process Specific Constants
cCubeSrc            = 'Source Cube';
cCubeTgt            = 'Target Cube';

#EndRegion Process Declarations
################################################################################################# 

### LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

################################################################################################# 
#Region Validate Parameters

# pLogOutput
If( pLogOutput >= 1 );
    pLogOutput = 1;
Else;
    pLogOutput = 0;
EndIf;
    
# pTemp
If( pTemp >= 1 );
    pTemp = 1;
Else;
    pTemp = 0;
EndIf;

# Validate source cube
If( Trim( cCubeSrc ) @= '' );
    nErrors = nErrors + 1;
    sMessage = 'No source cube specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( cCubeSrc ) = 0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'Invalid source cube specified: %cCubeSrc%.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate target cube
If( Trim( cCubeTgt ) @= '' );
    nErrors = nErrors + 1;
    sMessage = 'No target cube specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ElseIf( CubeExists( cCubeTgt ) = 0 );
    nErrors = nErrors + 1;
    sMessage = Expand( 'Invalid target cube specified: %cCubeTgt%.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If any parameters fail validation then set data source of process to null and go directly to epilog
If( nErrors > 0 );
    DataSourceType = 'NULL';
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    Else;
        ProcessBreak;
    EndIf;
EndIf;

################################################################################################# 
#EndRegion Validate Parameters

### If required switch transaction logging off (this should be done AFTER the escape/reject if parameters fail validation and BEFORE the zero out commences)
nCubeLogChanges = CubeGetLogChanges( cCubeTgt );
CubeSetLogChanges( cCubeTgt, 0 );


################################################################################################# 
#Region - ZeroOut

sProc       = '}bedrock.cube.data.clear';
# Set filter as per logic requirement of the ZeroOut
sFilter     = 'Dim1' | sDelimEleStart | 'El1' | sDelimDim | 'Dim2' | sDelimEleStart | 'El2';
nRet        = ExecuteProcess( sProc, 'pLogOutput', pLogOutput,
  'pStrictErrorHandling', pStrictErrorHandling,
	'pCube', cCubeTgt, 'pView', cViewClr, 'pFilter', sFilter,
	'pDimDelim', sDelimDim, 'pEleStartDelim', sDelimEleStart, 'pEleDelim', sDelimEle,
	'pTemp', pTemp 
);
If( nRet <> ProcessExitNormal() );
    nErrors = nErrors + 1;
    sMessage= 'Error in ZeroOut.';
    DataSourceType = 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

#EndRegion - ZeroOut
################################################################################################# 

################################################################################################# 
#Region - DataSource

sProc   = '}bedrock.cube.view.create';
# Set filter as per logic requirement of the data source processing
sFilter = Expand('Dim1%sDelimEleStart%El1%%sDelimDim%Dim2%sDelimEleStart%El2%');
# Adjust parameters for skipping of blanks / consols / rule calcs as required
bSuppressNull   = 1;
bSuppressC      = 1;
bSuppressRule   = 1;
nRet = ExecuteProcess( sProc, 'pLogOutput', pLogOutput, 
  'pStrictErrorHandling', pStrictErrorHandling,
	'pCube', cCubeSrc, 'pView', cViewSrc, 'pFilter', sFilter,
	'pSuppressZero', bSuppressNull, 'pSuppressConsol', bSuppressC, 'pSuppressRules', bSuppressRule,
	'pDimDelim', sDelimDim, 'pEleStartDelim', sDelimEleStart, 'pEleDelim', sDelimEle,
	'pTemp', pTemp
);
If( nRet <> ProcessExitNormal() );
    nErrors = nErrors + 1;
    sMessage= 'Error in source view creation.';
    DataSourceType = 'NULL';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

### Assign data source
If( nErrors = 0 );
    DatasourceType          = 'VIEW';
    DatasourceNameForServer = cCubeSrc;
    DatasourceCubeView      = cViewSrc;
EndIf;

#EndRegion - DataSource
################################################################################################# 

### End Prolog ###
#endregion
#region Metadata
#****Begin: Generated Statements***
#****End: Generated Statements****

If( pLogOutput >= 1 );
   nMetadataRecordCount = nMetadataRecordCount + 1;
EndIf;
#endregion
#region Data
#****Begin: Generated Statements***
#****End: Generated Statements****

If( pLogOutput >= 1 );
   nDataRecordCount = nDataRecordCount + 1;
EndIf;
#endregion
#region Epilog
################################################################################################# 
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0~~##
################################################################################################# 

#****Begin: Generated Statements***
#****End: Generated Statements****

### If required switch transaction logging back on 
CubeSetLogChanges( cCubeTgt, nCubeLogChanges );

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully %cAction%. %nDataRecordCount% records processed.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion