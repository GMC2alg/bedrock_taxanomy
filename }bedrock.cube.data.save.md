#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.cube.data.save', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pCube', '', 'pDelim', '&'
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
# This process save data to disk for the cubes provided in parameter.

# Use case: Intended for Development or production.
#1/ This process would be used any time data for a specific cube need to be saved (i.e.: After a data loading to save a specific cube or or for manual entry cubes).

# Note:
# Naturally, a valid  cube name (pCube) is mandatory otherwise the process will abort. Wildcards and lists are acceptable.
# This process will save data for a cube.
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
sMessage = 	'';
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pCube:%pCube%, pDelim:%pDelim%.' ;  

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

nErrors = 0;

### Validate Parameters ###
# If blank delimiter specified then convert to default
If( pDelim @= '' );
  pDelim = '&';
EndIf;

# If no cubes have been specified then terminate process
If( Trim( pCube ) @= '' );
  sMessage = 'No cubes specified';
  nErrors = nErrors + 1;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;


### SET DATA SOURCE ###

DatasourceType = 'NULL';


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
      # If it has then search the relevant Cube folder to find the matches
      If( Scan( '*', sCube ) = 0 );
        If( CubeExists( sCube ) = 1 ); 
          CubeSaveData( sCube );
        Endif;
      Else;
        # Create subset of cubes using Wildcard
        sCubeExp = '"'|sCube|'"';
        sMdxPart = '{TM1FILTERBYPATTERN( TM1SUBSETALL( [}Cubes] ) ,'| sCubeExp | ')}';
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
            # Save data
            CubeSaveData( sCurrCube );
          Endif;
            nCountCubes = nCountCubes - 1;
        End;
      EndIf;

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully saved data for cube %pCube% .' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;


### End Epilog ###
#endregion