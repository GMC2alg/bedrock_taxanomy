#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.hier.sub.create.orphans', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pDim', '', 'pHier', '', 'pTemp', 1
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
# This process will create a static subset in a Hierarchy of target Dimension that consists of
# all orphan elements.

# Note:
# Orphan element is defined as:
# - Consolidated element without children.
# - Leaf element without parent.
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
cLogInfo            = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pHier:%pHier%, pTemp:%pTemp%.'; 
cAttributeDim       = '}ElementAttributes_' | pDim;
cSubsetOrphanC = 'Orphan C Elements (no children)';
cSubsetOrphanN = 'Orphan N Elements (no parents)';

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

IF(HierarchyExists(pDim, sHier ) = 0 );
  nErrors = 1;
  sMessage = 'Invalid dimension Hierarchy: ' | pDim |':'|sHier;
  LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

## Validate pTemp
IF( pTemp <> 0 & pTemp <> 1 );
    nErrors = 1;
    sMessage = 'Wrong parameter pTemp value (only 0 or 1 accepted).';
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
EndIf;

### Create Subsets ###
If( nErrors = 0 );
  If( HierarchySubsetExists( pDim, sHier, cSubsetOrphanC ) = 1 );
    HierarchySubsetDeleteAllElements( pDim, sHier, cSubsetOrphanC );
  Else;
    HierarchySubsetCreate( pDim, sHier, cSubsetOrphanC, pTemp );
  EndIf;
  If( HierarchySubsetExists( pDim, sHier, cSubsetOrphanN ) = 1 );
    HierarchySubsetDeleteAllElements( pDim, sHier, cSubsetOrphanN );
  Else;
    HierarchySubsetCreate( pDim, sHier, cSubsetOrphanN, pTemp );
  EndIf;
EndIf;


### Populate subsets ###
nElementCount = DimSiz( pDim|':'|sHier);
nElementIndex = 1;
nLeafCount = 0;
nConsolCount = 0;
While( nElementIndex <= nElementCount );
  sElement = ElementName( pDim, sHier, nElementIndex );
  If( ElementType( pDim, sHier, sElement ) @= 'N' & ElementParent( pDim, sHier, sElement, 1 ) @= '' );
    # N element with no parents
    nLeafCount = nLeafCount + 1;
    HierarchySubsetElementInsert( pDim, sHier, cSubsetOrphanN, sElement, nLeafCount );
  EndIf;
  If(ElementType(pDim,sHier, sElement) @= 'C' & ElementComponentCount(pDim, sHier, sElement) = 0);
    # C element with no children
    nConsolCount = nConsolCount + 1;
    HierarchySubsetElementInsert( pDim, sHier, cSubsetOrphanC, sElement, nConsolCount );
  EndIf;
  nElementIndex = nElementIndex + 1;
End;


### Tidy up ###

# If no orphans then destroy empty subsets
If( nErrors = 0 );
  If( HierarchySubsetGetSize( pDim, sHier, cSubsetOrphanN ) = 0 );
    HierarchySubsetDestroy( pDim, sHier, cSubsetOrphanN );
  EndIf;
  If( HierarchySubsetGetSize( pDim, sHier, cSubsetOrphanC ) = 0 );
    HierarchySubsetDestroy( pDim, sHier, cSubsetOrphanC );
  EndIf;
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
    If( nLeafCount > 0 );
       sProcessAction = Expand( 'Process:%cThisProcName% successfully created subset %cSubsetOrphanN% from dimension %pDim%:%pHier%.' );
       sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
       nProcessReturnCode = 1;
       If( pLogoutput = 1 );
           LogOutput('INFO', Expand( sProcessAction ) );
           nProcessReturnCode = 0; 
       EndIf;
    EndIf ;
    
    If( nConsolCount > 0 );
       sProcessAction = Expand( 'Process:%cThisProcName% successfully created subset %cSubsetOrphanC% from dimension %pDim%:%pHier%.' );
 
       sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
       nProcessReturnCode = 1;

      If( pLogoutput = 1 );
          LogOutput('INFO', Expand( sProcessAction ) );   
      EndIf;
    Endif ;
EndIf;

### End Epilog ###
#endregion