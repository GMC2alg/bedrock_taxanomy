#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.exclude', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
    	'pDim', '', 'pHier', '', 'pSub', '',
    	'pExclusions', '', 'pDelim', '&', 'pTemp', 1
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
# This process will remove specified elements from a subset in a Hierarchy of target Dimension.
# Wildcard characters `*`and `?` are accepted in list of elements to be excluded.

# Note:
# - If a leaf level element is specified, it will be removed on its own.
# - If a consolidated element is specified it will be removed as well as its descendants.

# Caution: Target hierarchy cannot be `Leaves`.
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName       = GetProcessName();
cUserName           = TM1User();
cTimeStamp          = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt          = NumberToString( INT( RAND( ) * 1000 ));
cTempSub            = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel      = 'ERROR';
cMsgErrorContent    = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cLogInfo            = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pSub:%pSub%, pExclusions:%pExclusions%, pDelim:%pDelim%, pTemp:%pTemp%.'; 
cAttributeDim       = '}ElementAttributes_' | pDim;

## LogOutput parameters
IF ( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;


### Validate Parameters ###

nErrors = 0;

If( Scan( ':', pDim ) > 0 & pHier @= '' );
    # A hierarchy has been passed as dimension. Handle the input error by splitting dim:hier into dimension & hierarchy
    pHier       = SubSt( pDim, Scan( ':', pDim ) + 1, Long( pDim ) );
    pDim        = SubSt( pDim, 1, Scan( ':', pDim ) - 1 );
EndIf;

# Validate dimension
If( Trim( pDim ) @= '' );
  nErrors = 1;
  sMessage = 'No dimension specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
If( DimensionExists( pDim ) = 0 );
  nErrors = 1;
  sMessage = 'Invalid dimension: ' | pDim;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate Hierarchy

IF(pHier @= 'Leaves' );
  nErrors = 1;
  sMessage = 'Invalid  Hierarchy: ' | pDim |':'|pHier;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

If( Trim( pHier ) @= '' );
  sHier = pDim;
Else;
  sHier = pHier;
EndIf;

If( HierarchyExists( pDim, sHier ) = 0 );
  nErrors = 1;
  sMessage = 'The Hierachy ' | sHier | ' does not exists.';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate subset
If( Trim( pSub ) @= '' );
  nErrors = 1;
  sMessage = 'No subset specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;
If( HierarchySubsetExists( pDim,sHier, pSub ) = 0 );
  nErrors = 1;
  sMessage = 'Invalid subset: ' | pSub | ' in dimension:Hierarchy ' | pDim |':' | sHier;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate Elements
If( Trim( pExclusions ) @= '' );
  nErrors = 1;
  sMessage = 'No Elements specified';
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

# Validate delimiter
If( pExclusions @<> '' & pDelim @= '' );
  pDelim = '&';
EndIf;

## Validate pTemp
IF( pTemp <> 0 & pTemp <> 1 );
    nErrors = 1;
    sMessage = 'Wrong parameter pTemp value (only 0 or 1 accepted).';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

### Process Elements ###
If(nErrors = 0);
  nDelimIndex = 1;
  sExclusions = pExclusions;

  While( nDelimIndex <> 0 & Long( sExclusions ) > 0 );

    nDelimIndex = Scan( pDelim, sExclusions );
    If( nDelimIndex <> 0 );
      sExclusion = Trim( SubSt( sExclusions, 1, nDelimIndex - 1 ) );
      sExclusions = Trim( SubSt( sExclusions, nDelimIndex + Long( pDelim ), Long( sExclusions ) ) );
    Else;
      sExclusion = Trim( sExclusions );
    EndIf;
    If(Scan('*',sExclusion) = 0 & Scan('?',sExclusion) = 0);
      # Check that Element is present in the dimension
      If( ElementIndex ( pDim, sHier, sExclusion ) <> 0 );
        sExclusion = HierarchyElementPrincipalName( pDim, sHier, sExclusion );
        # Work through subset and remove Element
        nSubsetIndex = 1;
        nSubsetSize = HierarchySubsetGetSize( pDim, sHier, pSub );
        While( nSubsetIndex <= nSubsetSize );
          sTemp = HierarchySubsetElementGetIndex (pDim, sHier, pSub, '', nSubsetIndex);
          sElement = HierarchySubsetGetElementName( pDim, sHier, pSub, nSubsetIndex );
          # If Element is found or a descendant of the Element is found the remove from subset
          If( sElement @= sExclusion % ElementIsAncestor( pDim, sHier, sExclusion, sElement ) = 1 );
            sTemp = HierarchySubsetElementGetIndex (pDim, sHier, pSub, '', nSubsetIndex);
            HierarchySubsetElementDelete ( pDim, sHier, pSub, nSubsetIndex );
            nSubsetSize = nSubsetSize - 1;
          Else;
            nSubsetIndex = nSubsetIndex + 1;
          EndIf;
        End;
      
      EndIf;
    Else;
      # Wildcard search string
        sExclusion = '"'|sExclusion|'"';
        stempSub = cThisProcName| cRandomInt;
        sProc = '}bedrock.hier.sub.create.bymdx';
        sMdx = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |pDim|':'|sHier |' ])},'| sExclusion| ')}';
        ExecuteProcess(sProc,
                      'pStrictErrorHandling', pStrictErrorHandling,
                    	'pDim',pDim,
                    	'pHier',sHier,
                    	'pSub',stempSub,
                    	'pMDXExpr',sMdx,
                    	'pConvertToStatic',1,
                    	'pTemp', pTemp);
        nSubsetindex = 1;
        nSubsetSize = HierarchySubsetGetSize(pDim, sHier, stempSub);
        While (nSubsetindex <= nSubsetSize);
          sTemp = HierarchySubsetElementGetIndex (pDim, sHier, stempSub, '', nSubsetIndex);
          sElement = HierarchySubsetGetElementName(pDim, sHier, stempSub, nSubsetindex);
          HierarchySubsetElementDelete( pDim, sHier,stempSub,nSubsetindex );
          nSubsetSize = nSubsetSize -1;
          ## Delete Element from main subset
          If(HierarchySubsetElementExists(pDim, sHier, pSub, sElement)>0);
            nSearchIndex = 1;
            nSearchSize = HierarchySubsetGetSize(pDim, sHier, pSub);
            While( nSearchIndex <= nSearchSize  );
              sSearchElement = HierarchySubsetGetElementName( pDim, sHier, pSub, nSearchIndex );
               # If Element is found or a descendant of the Element is found the remove from subset
                If( sElement @= sSearchElement % ElementIsAncestor( pDim, sHier, sElement, sSearchElement ) = 1 );
                  sTemp = HierarchySubsetElementGetIndex (pDim, sHier, pSub, '', nSearchIndex);
                  HierarchySubsetElementDelete ( pDim, sHier, pSub, nSearchIndex );
                  nSearchSize = 0;
                Else;
                  nSearchIndex = nSearchIndex + 1;
                EndIf;
            End;
          Endif;
          ######
        End;
        HierarchySubsetDestroy(pDim, sHier,stempSub);
    EndIf;

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
    sProcessAction = Expand( 'Process:%cThisProcName% successfully excluded elements from subset %pSub% from dimension %pDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion