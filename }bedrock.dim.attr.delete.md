#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.dim.attr.delete', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pDim', '', 'pAttr', '', 'pDelim', '&', 'pCtrlObj', 0
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
# This process can delete one or more attributes in one or more specified dimensions.
#
# Use case: Intended for development/prototyping.
# 1. Clean up unused dimension attributes before going to production
#
# Note:
# * Delimited lists and/or wild card(*) are acceptable for pDim & pAttr
# * Multi-character wildcard "*" value for pAttr is evaluated as "ALL"
# * You cannot specify "*" for **both** pDim and pAttr!
#
# Warning:
# 1. As the process accepts wildcards USE WITH GREAT CARE! As if using wildcards for dimensions and attributes any matching attributes 
# in any matching dimensions will be removed from the system.
# 2. Multi-character wildcard "*" value for pAttr is evaluated as "ALL". Setting pAttr to "*" will delete all attributes in the spacified dimension.
#EndRegion @DOC

### Global Variables
StringGlobalVariable('sProcessReturnCode');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode= 0;

### Constants ###
cThisProcName     = GetProcessName();
cUserName         = TM1User();
cTimeStamp        = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt        = NumberToString( INT( RAND( ) * 1000 ));
cTempSub          = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel    = 'ERROR';
cMsgErrorContent  = 'User:%cUserName% Process:%cThisProcName% ErrorMsg:%sMessage%';
cMsgInfoContent   = 'User:%cUserName% Process:%cThisProcName% Message:%sMessage%';
cLogInfo          = 'Process:%cThisProcName% run with parameters pDim:%pDim%, pAttr:%pAttr%, pDelim:%pDelim%, pCtrlObj:%pCtrlObj%.'; 

## LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );   
ENDIF;

### Validate Parameters ###
nErrors = 0;

## Validate dimension
If( (Trim( pDim ) @= 'ALL' & Trim( pAttr ) @= 'ALL') % (Trim( pDim ) @= '*' & Trim( pAttr ) @= 'ALL') % (Trim( pDim ) @= 'ALL' & Trim( pAttr ) @= '*') % (Trim( pDim ) @= '*' & Trim( pAttr ) @= '*') );
    nErrors = 1;
    sMessage = 'Deleting all attrbitutes from all dimensions is not supported.';
    LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
EndIf;

If( Trim( pDim ) @= '' );
    nErrors = 1;
    sMessage = 'No dimension specified.';
    LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
EndIf;

# Validate attribute
If( Trim( pAttr ) @= '' );
    nErrors = 1;
    sMessage = 'No attribute specified.';
    LogOutput( 'ERROR', Expand( cMsgErrorContent ) );
EndIf;

# If blank delimiter specified then convert to default
If( pDelim @= '' );
    pDelim = '&';
EndIf;

### Check for errors before continuing
If( nErrors > 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;

# Loop through dimensions in pDim and attributes in pAttr
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
    IF( pCtrlObj = 1 );
      sMdxPart = '{TM1FILTERBYPATTERN( EXCEPT( TM1SUBSETALL( [}Dimensions] ) , TM1FILTERBYPATTERN( TM1SUBSETALL( [}Dimensions] ) , "*:*") ) ,'| sDimExp | ')}';
    ELSE;
      sMdxPart = '{TM1FILTERBYPATTERN( EXCEPT( EXCEPT( TM1SUBSETALL( [}Dimensions] ) , TM1FILTERBYPATTERN( TM1SUBSETALL( [}Dimensions] ) , "*:*") ) , TM1FILTERBYPATTERN( TM1SUBSETALL( [}Dimensions] ) , "}*" ) ) ,'| sDimExp | ')}';  
    ENDIF;
    IF( sMdx @= ''); 
      sMdx = sMdxPart; 
    ELSE;
      sMdx = sMdx | ' + ' | sMdxPart;
    ENDIF;
    
    If( SubsetExists( '}Dimensions' , cTempSub ) = 1 );
        # If a delimited list of dim names includes wildcards then we may have to re-use the subset multiple times
        SubsetMDXSet( '}Dimensions' , cTempSub, sMDX );
    Else;
        # temp subset, therefore no need to destroy in epilog
        SubsetCreatebyMDX( cTempSub, sMDX, '}Dimensions' , 1 );
    EndIf;
    
    # Loop through dimensions in subset created based on wildcard
    nCountDim = SubsetGetSize( '}Dimensions' , cTempSub );
    While( nCountDim >= 1 );
        sDim = SubsetGetElementName( '}Dimensions' , cTempSub, nCountDim );
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
            # Loop through attributes in pAttr 
            sAttrs              = pAttr;
            nDelimiterIndexA    = 1;
            sAttrDim            = '}ElementAttributes_'|sDim ;
            sMdxAttr = '';
            While( nDelimiterIndexA <> 0 );

                nDelimiterIndexA = Scan( pDelim, sAttrs );
                If( nDelimiterIndexA = 0 );
                    sAttr   = sAttrs;
                Else;
                    sAttr   = Trim( SubSt( sAttrs, 1, nDelimiterIndexA - 1 ) );
                    sAttrs  = Trim( Subst( sAttrs, nDelimiterIndexA + Long(pDelim), Long( sAttrs ) ) );
                EndIf;

                # Validate if sDim has attributes
                IF( DimensionExists( '}ElementAttributes_'| sDim ) = 0 );
                    sMessage = 'Dimension ' | sDim | ' has no attributes.';
                    LogOutput( 'INFO' , Expand( cMsgInfoContent ) );
                ElseIf( sAttr @= '*' );
                    # Delete attribute cube and dimension if pAttr is blank or set to ALL
                    CubeDestroy( sAttrDim );
                    DimensionDestroy( sAttrDim );
                Else;
                    # Create subset of attributes using Wildcard to loop through attributes in pAttr with wildcard
                    sAttr = '"'|sAttr|'"';
                    sMdxAttrPart = '{TM1FILTERBYPATTERN( {TM1SUBSETALL([ ' |sAttrDim| '])},'| sAttr| ')}';
                    IF( sMdxAttr @= ''); 
                      sMdxAttr = sMdxAttrPart; 
                    ELSE;
                      sMdxAttr = sMdxAttr | ' + ' | sMdxAttrPart;
                    ENDIF;
                    If( SubsetExists( sAttrDim, cTempSub ) = 1 );
                        # If a delimited list of attr names includes wildcards then we may have to re-use the subset multiple times
                        SubsetMDXSet( sAttrDim, cTempSub, sMdxAttr );
                    Else;
                        # temp subset, therefore no need to destroy in epilog
                        SubsetCreatebyMDX( cTempSub, sMdxAttr, sAttrDim, 1 );
                    EndIf;
                
                    # Loop through subset of attributes created based on wildcard
                    nCountAttr = SubsetGetSize( sAttrDim, cTempSub );
                    While( nCountAttr >= 1 );
                        sAttr = SubsetGetElementName( sAttrDim, cTempSub, nCountAttr );
                        # Validate attribute name in sDim
                        If( Dimix( sAttrDim , sAttr ) = 0 );
                            sMessage = Expand('The %sAttr% attribute does NOT exist in the %sDim% dimension.');
                            LogOutput( 'INFO' , Expand( cMsgInfoContent ) );
                        Else;
                            AttrDelete( sDim , sAttr ) ;
                            If( pLogOutput = 1 );
                                LogOutput( 'INFO', Expand( 'Attribute "%sAttr%" deleted from dimension %sDim%.' ) );
                            EndIf;
                        Endif;
                        nCountAttr = nCountAttr - 1;
                    End;
                Endif;
                
            End; 
        EndIf;
        
        nCountDim = nCountDim - 1;
    End;
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
    sProcessAction      = Expand( 'Process:%cThisProcName% successfully deleted attribute(s) %pAttr% from dimension(s) %pDim%.' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion