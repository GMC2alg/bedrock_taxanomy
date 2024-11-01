#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.element.delete', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '', 'pEle', '',
    	'pDelim', '&'
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
# This process will delete specified or all elements from a dimension Hierarchy. Elements might be
# specified as a delimited list of elements. Each member in the list might be specified exactly or
# by a wildcard pattern. Wildcards "\*" and "?" are accepted.
#
# Caution: When pEle is set to \*, __all__ elements in pHier will be deleted!
#EndRegion @DOC

##Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName     = GetProcessName();
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cTempSubDim       = cThisProcName |'_dims_'| cTimeStamp |'_'| cRandomInt;
cTempSubHier      = cThisProcName |'_hiers_'| cTimeStamp |'_'| cRandomInt;
cTempSubEle       = cThisProcName |'_eles_'| cTimeStamp |'_'| cRandomInt;
cUserName         = TM1User();
cMsgErrorLevel    = 'ERROR';
cMsgInfoLevel     = 'INFO';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cMsgInfoContent   = 'User:%cUserName% Process:%cThisProcName% Message:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pEle:%pEle%, pDelim:%pDelim%.'; 

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

If( Scan( '*', pDim ) = 0 & Scan( '?', pDim ) = 0 & Scan( pDelim, pDim ) = 0 & Scan( ':', pDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pDim, Scan( ':', pDim ) + 1, Long( pDim ) );
    pDim        = SubSt( pDim, 1, Scan( ':', pDim ) - 1 );
EndIf;

# Validate Dimension
If( Trim( pDim ) @= '' );
    nErrors = 1;
    sMessage = 'No dimension specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# If blank delimiter specified then convert to default
If( pDelim @= '' );
    pDelim = '&';
EndIf;

# Validate Hierarchy
If( Trim( pHier ) @= '' );
    ## use same name as Dimension. Since wildcards are allowed this is managed inside the code below
EndIf;

### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

# Loop through dimensions in pDim
sDims = pDim;
nDimDelimiterIndex = 1;
sMdx = '';
# Get 1st dimension
While( nDimDelimiterIndex <> 0 );
    # Extract 1st dimension > sDim
    nDimDelimiterIndex = Scan( pDelim, sDims );
    If( nDimDelimiterIndex = 0 );
        sDim = sDims;
    Else;
        sDim = Trim( SubSt( sDims, 1, nDimDelimiterIndex - 1 ) );
        sDims = Trim( Subst( sDims, nDimDelimiterIndex + Long(pDelim), Long( sDims ) ) );
    EndIf;
    
      # Create subset of dimensions using Wildcard to loop through dimensions in pDim with wildcard
    sDimExp = '"'|sDim|'"';
    sMdxPart = '{TM1FILTERBYPATTERN( EXCEPT( TM1SUBSETALL( [}Dimensions] ) , TM1FILTERBYPATTERN( TM1SUBSETALL( [}Dimensions] ) , "*:*") ) ,'| sDimExp | ')}';
    IF( sMdx @= ''); 
      sMdx = sMdxPart; 
    ELSE;
      sMdx = sMdx | ' + ' | sMdxPart;
    ENDIF;
End;

If( SubsetExists( '}Dimensions' , cTempSubDim ) = 1 );
    # If a delimited list of dim names includes wildcards then we may have to re-use the subset multiple times
    SubsetMDXSet( '}Dimensions' , cTempSubDim, sMDX );
Else;
    # temp subset, therefore no need to destroy in epilog
    SubsetCreatebyMDX( cTempSubDim, sMDX, '}Dimensions' , 1 );
EndIf;

# Loop through dimensions in subset created based on wildcard
nCountDim = SubsetGetSize( '}Dimensions' , cTempSubDim );
While( nCountDim >= 1 );
    sDim = SubsetGetElementName( '}Dimensions' , cTempSubDim, nCountDim );
    # Validate dimension name
    If( DimensionExists(sDim) = 0 );
        nErrors = 1;
        sMessage = Expand( 'Dimension %sDim% does not exist.' );
        LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
    Else;
        If( pLogOutput = 1 );
          sMessage = Expand( 'Dimension %sDim% being processed....' );
          LogOutput( 'INFO', Expand( cMsgInfoContent ) );
        EndIf;
        # Loop through hierarchies in pHier
        If( Trim( pHier ) @= '' );
          ### Use main hierarchy for each dimension if pHier is empty
          sHierarchies      = sDim;
        Else;
          sHierarchies      = pHier;
        EndIf;
        nDelimiterIndexA    = 1;
        sHierDim            = '}Dimensions' ;
        sMdxHier            = '';
        While( nDelimiterIndexA <> 0 );

            nDelimiterIndexA = Scan( pDelim, sHierarchies );
            If( nDelimiterIndexA = 0 );
                sHierarchy   = sHierarchies;
            Else;
                sHierarchy   = Trim( SubSt( sHierarchies, 1, nDelimiterIndexA - 1 ) );
                sHierarchies = Trim( Subst( sHierarchies, nDelimiterIndexA + Long(pDelim), Long( sHierarchies ) ) );
            EndIf;

            # Create subset of Hierarchies using Wildcard
            If( sHierarchy  @= sDim );
                sHierExp    = '"'|sDim|'"';
            Else;
                sHierExp    = '"'|sDim|':'|sHierarchy|'"';
            EndIf;
            sMdxHierPart    = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sHierDim| '])},'| sHierExp | ')}';
            IF( sMdxHier    @= ''); 
              sMdxHier      = sMdxHierPart; 
            ELSE;
              sMdxHier      = sMdxHier | ' + ' | sMdxHierPart;
            ENDIF;
        End;
        If( Trim( pHier )   @= '*' );
          sMdxHier          = '{ UNION ( ' | sMdxHier |' , {[}Dimensions].[' | sDim | ']} )}';
        EndIf;

        If( SubsetExists( sHierDim, cTempSubHier ) = 1 );
            # If a delimited list of hier names includes wildcards then we may have to re-use the subset multiple times
            SubsetMDXSet( sHierDim, cTempSubHier, sMdxHier );
        Else;
            # temp subset, therefore no need to destroy in epilog
            SubsetCreatebyMDX( cTempSubHier, sMdxHier, sHierDim, 1 );
        EndIf;
    
        # Loop through subset of hierarchies created based on wildcard
        nCountHier = SubsetGetSize( sHierDim, cTempSubHier );
        While( nCountHier >= 1 );
            sCurrHier = SubsetGetElementName( sHierDim, cTempSubHier, nCountHier );
            sCurrHierName = Subst( sCurrHier, Scan(':', sCurrHier)+1, Long(sCurrHier) );
            # Validate hierarchy name in sHierDim
            If( Dimix( sHierDim , sCurrHier ) = 0 );
                sMessage = Expand('The %sCurrHier% hierarchy does NOT exist in the %sDim% dimension.');
                LogOutput( 'INFO' , Expand( cMsgInfoContent ) );
            Else;
              If( pLogOutput = 1 );
                sMessage = Expand( 'Hierarchy %sCurrHierName% in Dimension %sDim% being processed....' );
                LogOutput( 'INFO', Expand( cMsgInfoContent ) );
              EndIf;
              # Loop through hierarchy elements in pEle
              sEles = pEle;
               nDelimiterIndexB = 1;
              While( nDelimiterIndexB <> 0 );
                  
                  nDelimiterIndexB = Scan( pDelim, sEles );
                  If( nDelimiterIndexB = 0 );
                      sEle = sEles;
                  Else;
                      sEle = Trim( SubSt( sEles, 1, nDelimiterIndexB - 1 ) );
                      sEles = Trim( Subst( sEles, nDelimiterIndexB + Long(pDelim), Long( sEles ) ) );
                  EndIf;
                  
                  # Check if a wildcard has been used to specify the Element name.
                  # If it hasn't then just delete the Element if it exists
                  If(sEle @= '*');
                          HierarchyDeleteAllElements(sDim, sCurrHierName);
                  ElseIf( Scan( '*', sEle ) = 0 & Scan( '?', sEle ) = 0);
                      If( HierarchyElementExists( sDim,sCurrHierName, sEle ) = 1 );
                          HierarchyElementDelete( sDim, sCurrHierName,sEle );
                          If( sCurrHierName @= 'Leaves' );
                              sMessage = Expand('Element %sEle% deleted from LEAVES hierarchy in dimension %sDim%. This action removes the element from all hierarchies!');
                              LogOutput( cMsgInfoLevel, Expand( cMsgInfoContent ) );
                          ElseIf( pLogOutput = 1 );
                              sMessage = Expand( 'Element %sEle% deleted from hierarchy %sCurrHierName% in dimension %sDim%.' );
                              LogOutput( cMsgInfoLevel, Expand( cMsgInfoContent ) );
                          EndIf;
                      Else;
                          If( pLogOutput >= 1 );
                              sMessage = Expand('The Hierarchy %sCurrHier% does not contain element %sEle%.');
                              LogOutput( cMsgInfoLevel, Expand( cMsgInfoContent ) );
                          EndIf;
                      Endif;
                  Else;
                      # Wildcard search string
                      sEle = '"'|sEle|'"';
                      sProc = '}bedrock.hier.sub.create.bymdx';
                      sMdxEle = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' | sCurrHier |' ])},'| sEle| ')}';

                      If( HierarchySubsetExists( sDim, sCurrHierName, cTempSubEle ) = 1 );
                          # If a delimited list of ele names includes wildcards then we may have to re-use the subset multiple times
                          HierarchySubsetMDXSet( sDim, sCurrHierName, cTempSubEle, sMDXEle );
                      Else;
                          # temp subset, therefore no need to destroy in epilog
                          SubsetCreatebyMDX( cTempSubEle, sMDXEle, sCurrHier, 1 );
                      EndIf;

                      # Loop through subset of hierarchy elements created based on wildcard
                      nCountElems = HierarchySubsetGetSize(sDim, sCurrHierName, cTempSubEle);
                      While( nCountElems >= 1 );
                          sElement = HierarchySubsetGetElementName(sDim, sCurrHierName, cTempSubEle, nCountElems);
                          HierarchyElementDelete( sDim, sCurrHierName,sElement );
                          If( pLogOutput = 1 );
                              sMessage = Expand( 'Element %sElement% deleted from hierarchy %sCurrHierName% in dimension %sDim%.' );
                              LogOutput( cMsgInfoLevel, Expand( cMsgInfoContent ) );
                          EndIf;
                          nCountElems = nCountElems - 1;
                      End;
                  EndIf;
              
              End;
          Endif;
          
            nCountHier = nCountHier - 1;
        End;
            
    EndIf;
    
    nCountDim = nCountDim - 1;
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
    sProcessAction      = Expand( 'Process:%cThisProcName% successfully deleted the appropriate elements in hierarchy %pDim%:%pHier%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion