#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.delete', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '', 'pSub', '',
    	'pDelim', '&', 'pMode', 0
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
# This process deletes public subsets from selected dimension and hierarchies. Dimensions, hierarchies
# and subsets can be specified as a delimited list. Wildcards are accepted as `*` and `?` characters.

# Note:
# - pDim: To specify which dimensions to delete subsets from use the pDim parameter.
#   - To delete subsets from a single dimension only just specify that dimension name e.g. `Product`.
#   - To delete subsets from multiple dimensions specify each dimension name separated by a delimiter
#     e.g. `Product & Customer & Account`.
#   - To delete subsets from all dimensions then leave pDim parameter blank or supply `ALL`.
#   - If any invalid dimensions are specified they will be skipped but the process will continue to 
#     process the other dimensions.
#   - When specifying dimension names wildcards are permitted and all dimensions that match the 
#     wildcard search string will be searched - for syntax see below pSub parameter.
# - pHier: To specify which hierarchies to delete subsets from use the pHier parameter.
#   - To specify default hierachy, leave parameter value empty.
#   - To specify all available hierarchies, set parameter value to `*`.
#   - When specifying hierarchy names wildcards are permitted and all hierachies that match the wildcard 
#     search string will be searched - for syntax see below pSub parameter.
# - pSub: To specify which subsets to delete use the pSub parameter.
#   - This parameter must be specified, a blank value will cause the process to terminate.
#   - To delete a single subset only just specify that subset name e.g. `SmallProducts`.
#   - To delete multiple subsets specify each subset name separated by a delimiter
#     e.g. `SmallProducts & LargeProducts`.
# - pDelim: The delimiter is used when specifying multiple dimensions and/or multiple subsets.
#   - The default delimiter is `&`.
#   - Any delimiter can be used by specifying a value for pDelim.
#   - Choose a delimiter that won't be used in either the wildcard search strings or dimension names.
# - pMode:
#   - When set to value less or equal to `1`: all found subsets will be destroyed.
#   - When set to value of `2`: all found subsets will be emptied.  
# __Example__:
# - When specifying subset names wildcards are permitted and all subsets that match the wildcard 
#   search string will be deleted:
#     - The wildcard search string follows the same format as wildcards used in Subset Editor.
#     - To delete all subsets that __start__ with a specific string use a trailing `*` e.g. `Bedrock*`.
#     - To delete all subsets that __end__ in a specific string use a leading `*` e.g. `*Bedrock`.
#     - To delete all subsets that __contain__ a specific string use leading and trailing `*`'s e.g. `*Bedrock*`.
#     - To delete a single specific subset only don't use `*`'s at all e.g. `Bedrock`.
#     - To specify multiple search stings list them all separated by a delimiter e.g. `Bedrock* & *Temp & *Test*`.
#     - Similarly you can use `?` as wildcard specifying one character - usage is similar as above specified examples for `*`.
#     - Specific subset names and wildcard based names can both be used together e.g. `SalesByProduct;Bedrock*`
#EndRegion @DOC

### Global Variables

StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants

cThisProcName     = GetProcessName();
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cTempSub          = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cTempSubDim       = cThisProcName |'_Dim_'| cTimeStamp |'_'| cRandomInt;
cUserName         = TM1User();
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pSub:%pSub%, pMode:%pMode%.';
cAll              = 'ALL';
cDimDimensions    = '}Dimensions';
cCharAny          = '?';
cStringAny        = '*';
cCharDimHier      = ':';

### Flag - quit before parsing of parameters
nSkipParsing      = 0;
sProcessAction    = '';

### LogOutput parameters
IF( pLogoutput = 1 );
  LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters

nErrors = 0;

If( Scan( '*', pDim ) = 0 & Scan( '?', pDim ) = 0 & Scan( pDelim, pDim ) = 0 & Scan( ':', pDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pDim, Scan( ':', pDim ) + 1, Long( pDim ) );
    pDim        = SubSt( pDim, 1, Scan( ':', pDim ) - 1 );
EndIf;

### Validate delimiter
If( Trim( pDelim ) @= '' );
  pDelim = '&';
EndIf;

### Validate dimension
If( Trim( pDim ) @= '' );
  nErrors = 1;
  sMessage = 'No dimension specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;
If( SCAN( cCharAny, pDim ) = 0 & SCAN( cStringAny, pDim ) = 0 & SCAN( pDelim, pDim ) = 0 & SCAN( cCharDimHier, pDim ) > 0 & pHier @= '' );
    pHier = SubSt( pDim, SCAN( cCharDimHier, pDim ) + 1, Long( pDim ) );
    pDim = SubSt( pDim, 1, SCAN( cCharDimHier, pDim ) - 1 );
EndIf;

## Validate Hierarchy

IF(pHier @= 'Leaves' );
  nErrors = 1;
  sMessage = 'Invalid  Hierarchy: ' | pDim |':'|pHier;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( Trim( pHier ) @<> '' );
  sHier = pHier;
EndIf;

### Validate subset
If( Trim( pSub ) @= '' );
  nErrors = 1;
  sMessage = 'No subsets specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

### Destroy subset if exactly specified in parameters - important for recursive calls
If ( SCAN( cCharAny, pDim ) = 0 & SCAN( cStringAny, pDim ) = 0 & SCAN( pDelim, pDim ) = 0 &
     SCAN( cCharAny, pHier ) = 0 & SCAN( cStringAny, pHier ) = 0 & SCAN( pDelim, pHier ) = 0 &
     SCAN( cCharAny, pSub ) = 0 & SCAN( cStringAny, pSub ) = 0 & SCAN( pDelim, pSub ) = 0 );
  If ( DimensionExists( pDim ) = 0 );
    nErrors = 1;
    sMessage = Expand( 'Dimension %pDim% doesn''t exist.' );
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    Else;
        ProcessBreak;
    EndIf;
  EndIf;
  If ( pHier @<> '' & HierarchyExists( pDim, pHier ) = 0 );
    nErrors = 1;
    sMessage = Expand( 'Hierarchy %pHier% doesn''t exist in dimension %pDim%.' );
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    Else;
        ProcessBreak;
    EndIf;
  EndIf;
  If ( HierarchySubsetExists( pDim, pHier, pSub ) = 0 );
    nErrors = 1;
    sMessage = Expand( 'Subset %pSub% doesn''t exist in hierarchy %pHier% of dimension %pDim%.' );
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
    If( pStrictErrorHandling = 1 ); 
        ProcessQuit; 
    Else;
        ProcessBreak;
    EndIf;
  EndIf;
  If ( pHier @= '' );
    pHier = pDim;
  EndIf;
  If ( pMode <= 1 );
    HierarchySubsetDestroy( pDim, pHier, pSub );
    sProcessAction = Expand( 'Destroyed subset %pSub% on dimension %pDim% in hierarchy %pHier%.' );
  ElseIf ( pMode = 2 );
    HierarchySubsetDeleteAllElements( pDim, pHier, pSub );
    sProcessAction = Expand( 'Deleted all elements from subset %pSub% on dimension %pDim% in hierarchy %pHier%.' );
  EndIf;
  ### We don't need to parse any parameters as this was exact match, so we will finish processing
  nSkipParsing = 1;
EndIf;

### Need to process parameters before cleaning
If ( nSkipParsing = 0 );
  
  ### Validate all dimension case
  If( Trim( pDim ) @= '*' );
    pDim = cAll;
  EndIf;
  
  ### Handle All dimensions or a dimension list
  ### We will exclude hierarchies in this step and will filter them in connection with subsets in later steps
  If ( TRIM( pDim ) @= cAll );
    sMDX = Expand( '{FILTER(TM1SUBSETALL([%cDimDimensions%]), INSTR([%cDimDimensions%].CurrentMember.Name, '':'' ) = 0 )}' );
  Else;
    sDimTokenizer = TRIM( pDim );
    sMDX = '';
    ### Loop and tokenize dimension list
    While ( sDimTokenizer @<> '' );
      nPos = SCAN( pDelim, sDimTokenizer );
      If ( nPos = 0 );
        nPos = LONG( sDimTokenizer ) + 1;
      EndIf;
      sSearchDim = TRIM( SUBST( sDimTokenizer, 1, nPos - 1 ) );
      If( sMDX @= '' );
        sMDX = Expand( '{TM1FILTERBYPATTERN({FILTER(TM1SUBSETALL([%cDimDimensions%]), INSTR([%cDimDimensions%].CurrentMember.Name, '':'' ) = 0 )}, "%sSearchDim%")}' );
      Else;
        sMDX = Expand( '%sMDX% + {TM1FILTERBYPATTERN({FILTER(TM1SUBSETALL([%cDimDimensions%]), INSTR([%cDimDimensions%].CurrentMember.Name, '':'' ) = 0 )}, "%sSearchDim%")}' );
      EndIf;
      ### Consume dimension and delimiter
      sDimTokenizer = TRIM( DELET( sDimTokenizer, 1, nPos + LONG( pDelim ) - 1 ) );
    End;
    sMDX = Expand( '{%sMDX%}' );
  EndIf;
  
  ### Create dimensions subset
  If ( SubsetExists( cDimDimensions, cTempSub ) = 1 );
    SubsetMDXSet( cDimDimensions, cTempSub, sMDX );
  Else;
    SubsetCreatebyMDX( cTempSub, sMDX, cDimDimensions, 1 );
  EndIf;
  nMaxDim = SubsetGetSize( cDimDimensions, cTempSub );
  
  ### Loop through dimensions
  nCurDim = 1;
  While ( nCurDim <= nMaxDim );
    sCurDim = SubsetGetElementName( cDimDimensions, cTempSub, nCurDim );
    sCurSubDim = Expand( '}Subsets_%sCurDim%' );
    ### We will lookup subsets in }Subsets_ dimension of current dim
    If ( DimensionExists( sCurSubDim ) <> 0 );
      If ( pHier @= '' );
        sHier = sCurDim;
      Else;
        sHier = pHier;
      EndIf;
      sHierTokenizer = TRIM( sHier );
      sMDX = '';
      ### Loop and tokenize hierarchies list
      While ( sHierTokenizer @<> '' );
        nPos = SCAN( pDelim, sHierTokenizer );
        If ( nPos = 0 );
          nPos = LONG( sHierTokenizer ) + 1;
        EndIf;
        sSearchHier = TRIM( SUBST( sHierTokenizer, 1, nPos - 1 ) );
        sSubTokenizer = TRIM( pSub );
        ### Loop and tokenize subset list
        While ( sSubTokenizer @<> '' );
          nPos1 = SCAN( pDelim, sSubTokenizer );
          If ( nPos1 = 0 );
            nPos1 = LONG( sSubTokenizer ) + 1;
          EndIf;
          sSearchSubset = TRIM( SUBST( sSubTokenizer, 1, nPos1 - 1 ) );
          If ( sSearchHier @<> '*' );
            If ( sSearchHier @= sCurDim );
              sSearchMDX = sSearchSubset;
            Else;
              sSearchMDX = Expand( '%sSearchHier%:%sSearchSubset%' );
            EndIf;
            If( sMDX @= '' );
              sMDX = Expand( '{TM1FILTERBYPATTERN(TM1SUBSETALL( [%sCurSubDim%] ), "%sSearchMDX%")}' );
            Else;
              sMDX = Expand( '%sMDX% + {TM1FILTERBYPATTERN(TM1SUBSETALL( [%sCurSubDim%] ), "%sSearchMDX%")}' );
            EndIf;
          Else;
            # We need to handle special case of * specified as hierarchy filter - this is interpreted as ALL hierarchies - otherwise we would be skipping default hierarchy
            sSearchMDX1 = Expand( '%sSearchHier%:%sSearchSubset%' );
            sSearchMDX2 = Expand( '%sSearchSubset%' );
            If( sMDX @= '' );
              sMDX = Expand( '{TM1FILTERBYPATTERN(TM1SUBSETALL( [%sCurSubDim%] ), "%sSearchMDX1%")} + {TM1FILTERBYPATTERN(TM1SUBSETALL( [%sCurSubDim%] ), "%sSearchMDX2%")}' );
            Else;
              sMDX = Expand( '%sMDX% + {TM1FILTERBYPATTERN(TM1SUBSETALL( [%sCurSubDim%] ), "%sSearchMDX1%")} + {TM1FILTERBYPATTERN(TM1SUBSETALL( [%sCurSubDim%] ), "%sSearchMDX2%")}' );
            EndIf;
          EndIf;
          ### Consume subset and delimiter
          sSubTokenizer = TRIM( DELET( sSubTokenizer, 1, nPos1 + LONG( pDelim ) - 1 ) );
        End;
        ### Consume hierarchy and delimiter
        sHierTokenizer = TRIM( DELET( sHierTokenizer, 1, nPos + LONG( pDelim )- 1 ) );
      End;
      ### Query the hierarchies/subsets
      If ( SubsetExists( sCurSubDim, cTempSub ) = 1 );
        SubsetMDXSet( sCurSubDim, cTempSub, sMDX );
      Else;
        SubsetCreatebyMDX( cTempSub, sMDX, sCurSubDim, 1 );
      EndIf;
      nMaxSubs = SubsetGetSize( sCurSubDim, cTempSub );
      nCurSubs = nMaxSubs;
      While ( nCurSubs >= 1 );
        sCurSubs = SubsetGetElementName( sCurSubDim, cTempSub, nCurSubs );
        nColPos = SCAN( ':', sCurSubs );
        If ( nColPos = 0 );
          sHierarchy = sCurDim;
          sSubset = sCurSubs;
        Else;
          sHierarchy = SUBST( sCurSubs, 1, nColPos - 1 );
          sSubset = SUBST( sCurSubs, nColPos + 1, LONG(sCurSubs) - nColPos ); 
        EndIf;
        If ( pMode <= 1 );
          ### Recursive call to consume process error in this process and not to broadcast it to the parent caller process
          ExecuteProcess( cThisProcName,
            'pStrictErrorHandling', pStrictErrorHandling,
            'pLogOutput', pLogOutput,
            'pDim', sCurDim,
            'pHier', sHierarchy,
            'pSub', sSubset,
            'pDelim', pDelim,
            'pMode', pMode
          );
        ElseIf ( pMode = 2 );
          ### No need for a recursive call as we suppose this call is not going to raise a process error - this will save processing time significantly
          HierarchySubsetDeleteAllElements( pDim, pHier, pSub );
          sProcessAction = Expand( 'Deleted all elements from subset %pSub% on dimension %pDim% in hierarchy %pHier%.' );
        EndIf;  
        nCurSubs = nCurSubs - 1;
      End;
    EndIf;
    nCurDim = nCurDim + 1;
  End;
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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully deleted subset %pSub% from dimension %pDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion