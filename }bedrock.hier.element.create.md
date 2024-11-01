#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.element.create', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
      'pDim', '', 'pHier', '', 'pEle', '',
      'pEleType', '', 'pInsertionPoint', '',
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
# This process will create new element in a dimension Hierarchy. More elements than one will be
# created if pEle is supplied with a delimited list of elements.
 
# Note:
# Valid dimension name (pDim) and element list are mandatory, otherwise the process will abort.
 
# Caution: When target hierarchy is `Leaves`, no consolidated elements will be created.
#EndRegion @DOC
 
### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;
 
### Constants ###
cThisProcName     = GetProcessName();
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cTempSubDim       = cThisProcName |'_dims_'| cTimeStamp |'_'| cRandomInt;
cTempSubHier      = cThisProcName |'_hiers_'| cTimeStamp |'_'| cRandomInt;
cUserName         = TM1User();
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cMsgInfoContent   = 'User:%cUserName% Process:%cThisProcName% Message:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pEle:%pEle%, pEleType:%pEleType%, pInsertionPoint:%pInsertionPoint%, pDelim:%pDelim%.'; 
 
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
 
# Validate dimension
If( Trim( pDim ) @= '' );
    nErrors = 1;
    sMessage = 'No dimension specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
 
# Validate Hierarchy
If( Trim( pHier ) @= '' );
    ## use same name as Dimension. Since wildcards are allowed this is managed inside the code below
EndIf;
 
# Validate element
If( Trim( pEle ) @= '' );
    nErrors = 1;
    sMessage = 'No element specified.';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
 
# Validate element type
If( pEleType @= '' );
    pEleType = 'N';
EndIf;
pEleType = Upper( pEleType );
If( pEleType @<> 'N' & pEleType @<> 'C' & pEleType @<> 'S' );
    nErrors = 1;
    sMessage = 'Invalid element type: ' | pEleType;
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
 
# If blank delimiter specified then convert to default
If( pDelim @= '' );
    pDelim = '&';
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
        sHierDim            = '}Dimensions';
        sMdxHier            = '';
        While( nDelimiterIndexA <> 0 );
 
            nDelimiterIndexA = Scan( pDelim, sHierarchies );
            If( nDelimiterIndexA = 0 );
                sHierarchy   = sHierarchies;
            Else;
                sHierarchy   = Trim( SubSt( sHierarchies, 1, nDelimiterIndexA - 1 ) );
                sHierarchies = Trim( Subst( sHierarchies, nDelimiterIndexA + Long(pDelim), Long( sHierarchies ) ) );
            EndIf;
             
            ## If no wildcard specified and current hierarchy does not exist in dimension, create it
            If( Scan( '*', sHierarchy ) = 0 & Scan( '?', sHierarchy ) = 0);
              If( HierarchyExists( sDim, sHierarchy ) = 0 );
                  HierarchyCreate( sDim, sHierarchy );
              EndIf;
            EndIf;
 
            # Create subset of Hierarchies using Wildcard
            If( sHierarchy  @= sDim );
                sHierExp    = '"'| sHierarchy |'"';
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
            # If a delimited list of attr names includes wildcards then we may have to re-use the subset multiple times
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
            # Validate hierarchy name in dimension
            If( Dimix( sHierDim , sCurrHier ) = 0 );
                sMessage = Expand('The %sCurrHier% hierarchy does NOT exist in the %sDim% dimension.');
                LogOutput( 'INFO' , Expand( cMsgInfoContent ) );
            Else;
              If( pLogOutput = 1 );
                sMessage = Expand( 'Hierarchy %sCurrHierName% in Dimension %sDim% being processed....' );
                LogOutput( 'INFO', Expand( cMsgInfoContent ) );
              EndIf;
               
              # Extract, validate & add elements
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
                   
                  # Add elements that don't already exist
                  If( ElementIndex( sDim, sCurrHierName, sEle ) <> 0 );
                      If( pLogOutput = 1 );
                        sMessage = Expand( 'Element ' | sEle | ' already exist in hierarchy ' | sCurrHier);
                        LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
                      EndIf;
                  Else;
                      # Validate Insertion point
                      IF( pInsertionPoint @<> '' );
                          If( ElementIndex( sDim, sCurrHierName, pInsertionPoint ) = 0 );
                              If( pLogOutput = 1 );
                                sMessage = Expand( 'Element insertion point ' | pInsertionPoint | ' does NOT exist in dimension:hierarchy ' | sCurrHier | '. Using '' '' as insertion point.' );
                                LogOutput( 'INFO', Expand( cMsgInfoContent ) );
                              EndIf;
                              sInsertionPoint = '';
                          Else;
                              sInsertionPoint = pInsertionPoint;
                          EndIf;
                      Else;
                          sInsertionPoint = pInsertionPoint;
                      EndIf;
                      # Leaves hier
                      If( sCurrHierName @= 'Leaves' & pEleType @= 'C' );
                          If( pLogOutput = 1 );
                            sMessage = Expand( 'Invalid element type for Leaves hierarchy in dimension %sDim%: ' | pEleType );
                            LogOutput( cMsgErrorLevel, Expand( cMsgInfoContent ) );
                          EndIf;
                      Else;
                          HierarchyElementInsert( sDim, sCurrHierName, sInsertionPoint , sEle , pEleType );    
                          If( pLogOutput = 1 );
                            sMessage = Expand( 'Element %sEle% created in hierarchy %sCurrHier% having type ' | pEleType );
                            LogOutput( 'INFO', Expand( cMsgInfoContent ) );
                          EndIf;
                      EndIf;
                  Endif;
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
    sProcessAction     = Expand( 'Process:%cThisProcName% inserted element(s) %pEle% with type %pEleType% into %pDim%:%pHier%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;


### End Epilog ###
#endregion