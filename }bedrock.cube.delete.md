#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.delete', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pCube', '', 'pDelim', '&',
    	'pCtrlObj', 0
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
# This process deletes cube(s).

# Use case: Intended for cleaning up after development/prototyping.
# 1\ Delete all cubes not needed after Go Live.

# Note:
# A list of cubes can be specified and/or wild cards can be used.
# Naturally valid cube name(s) must be specified otherwise the process will abort.
# By default (pCtrlObj) the process will not delete control cubes (i.e. attributes, security etc).
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###

cThisProcName = GetProcessName();
cUserName = TM1User();
cTimeStamp = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt = NumberToString( INT( RAND( ) * 1000 ));
cTempSub          = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pDelim:%pDelim%, pCtrlObj:%pCtrlObj%.' ;  

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###

nErrors = 0;

# If blank delimiter specified then convert to default
If( pDelim @= '' );
  pDelim = '&';
EndIf;

# If no cubes have been specified, then log error message
If( Trim( pCube ) @= '' );
  nErrors = 1;
  sMessage = 'No cubes specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

# If no wildcard and no delimiter, log error message if cubename is invalid 
If( Scan( pDelim, pCube ) = 0 & Scan( '*', pCube ) = 0  & Trim( pCube ) @<> '' & CubeExists( pCube ) = 0 ); 
  nErrors = 1;
  sMessage = 'Cubename ' | pCube | ' is invalid';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### Split parameter into individual cubes and delete ###
sCubes = pCube;
nDelimiterIndex = 1;
sMdx = '';

While( nDelimiterIndex <> 0 );
  nDelimiterIndex = Scan( pDelim, sCubes );
  If( nDelimiterIndex = 0 );
    sCube = sCubes;
  Else;
    sCube = Trim( SubSt( sCubes, 1, nDelimiterIndex - 1 ) );
    sCubes = Trim( Subst( sCubes, nDelimiterIndex + Long(pDelim), Long( sCubes ) ) );
  EndIf;
  
  # Check if a wildcard has been used to specify the Cube name.
  # If it hasn't then just delete the Cube if it exists
  # If it has then create cubes subset using Wildcard expression in Mdx
  If( Scan( '*', sCube ) = 0 );
    If( CubeExists( sCube ) = 1 ); 
      If(Subst(sCube,1,1) @= '}');
        If(pCtrlObj = 1);
          CubeDestroy( sCube );
        Endif;
      Else;
        CubeDestroy( sCube );
      Endif;
    Endif;
  Else;
      # Create subset of cubes using Wildcard
    sCubeExp = '"'|sCube|'"';
    IF( pCtrlObj = 1 );
      sMdxPart = '{TM1FILTERBYPATTERN( TM1SUBSETALL( [}Cubes] ) ,'| sCubeExp | ')}';
    ELSE;
      sMdxPart = '{TM1FILTERBYPATTERN( EXCEPT( TM1SUBSETALL( [}Cubes] ) , TM1FILTERBYPATTERN( TM1SUBSETALL( [}Cubes] ) , "}*" ) ) ,'| sCubeExp | ')}';  
    ENDIF;
    IF( sMdx @= ''); 
      sMdx = sMdxPart; 
    ELSE;
      sMdx = sMdx | ' + ' | sMdxPart;
    ENDIF;
    
    If( SubsetExists( '}Cubes' , cTempSub ) = 1 );
        # If a delimited list of cube names includes wildcards then we may have to re-use the subset multiple times
        SubsetMDXSet( '}Cubes' , cTempSub, sMDX );
    Else;
        # temp subset, therefore no need to destroy in epilog
        SubsetCreatebyMDX( cTempSub, sMDX, '}Cubes' , 1 );
    EndIf;
    
    # Loop through cubes in subset created based on wildcard
    nCountCubes = SubsetGetSize( '}Cubes' , cTempSub );
    While( nCountCubes >= 1 );
      sCurrCube = SubsetGetElementName( '}Cubes' , cTempSub, nCountCubes );
      # Validate cube name
      If( CubeExists( sCurrCube ) = 1 ); 
        # Destroy Cube
        If(Subst(sCube,1,1) @= '}');
          If(pCtrlObj = 1);
            CubeDestroy( sCurrCube );
          Endif;
        Else;
          CubeDestroy( sCurrCube );
        Endif;
      Endif;
        nCountCubes = nCountCubes - 1;
    End;
  EndIf;

End;


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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully deleted cube %pCube%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion