#region Prolog
#Region CallThisProcess
# A snippet of code provided as an example how to call this process should the developer be working on a system without access to an editor with auto-complete.
If( 1 = 0 );
    ExecuteProcess( '}bedrock.server.util.string.validate', 'pLogOutput', pLogOutput,
      'pStrictErrorHandling', pStrictErrorHandling,
	    'pInputString', '', 'pUndesirableFileSystem', '/|\>"<:?*', 'pUndesirable1st', Char(39) | '+-[]@!{}%',
	    'pChanges', '\,B Slash&/,F Slash&|, &-,Minus&+,Plus&>,greater than&<,less than',
	    'pReplaceIfNotFound', '_',
	    'pDelim', '&', 'pSeperator', ',', 'pMode', 3 
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
# This process will validate a string pInputString based on rules in pChanges and change or 
# eliminate characters to create a global variable sOutputString that can be used in the source TI.

# Note:
# - pInputString: This is the input string that needs to be validated based on file system 
#   limitations or undesirable 1st characters.

# - pUndesirableFileSystem: These are characters considered undesirable (even forbidden) in 
#   object/element names due to file system limitations of the operation system. 

# - pUndesirable1st: These are characters considered undesirable as 1st characters in object/element
#   names due to TM1 limitations.

# - pChanges: This string defines the rule of how to change undesirable characters. It can be made up
#   of many definitions delimited by pDelim (e.g. `&` which is not considered undesirable
#   anywhere). Each definition would contain a character considered undesirable and the desired 
#   character separatedby pSeperator (e.g. to change a `%` to Percentage and `"` to inches, it would
#   be `%,Percentage&",inches` if pDelim = `&` and pSeperator = `,`).

# - pReplaceIfNotFound: This is a catch all for characters listed in pUndesirableFileSystem or 
#   pUndesirable1st that don't have a rule in pChanges.

# - pDelim: This is a character that is used to seperate definitions in pChanges.

# - pSeperator: This is a character used to seperate the current and desired character within each
#   definition in pChanges.

# - pMode: This can be used to limit whether the TI looks at pUndesirableFileSystem or pUndesirable1st 
#   without having to delete the characters in those parameters.
#EndRegion @DOC

#Region # Variables & Constants
# Global Variables
StringGlobalVariable('sProcessReturnCode');
StringGlobalVariable('sOutputString');
NumericGlobalVariable('nProcessReturnCode');
nProcessReturnCode = 0;

# Constants 
cThisProcName   = GetProcessName();
cUserName       = TM1User();
cTimeStamp      = TimSt( Now, '\Y\m\d\h\i\s' );
cRandomInt      = NumberToString( INT( RAND( ) * 1000 ));
cSubset         = cThisProcName |'_'| cTimeStamp |'_'| cRandomInt;
cMsgErrorLevel  = 'ERROR';
cMsgErrorContent= '%cThisProcName% : %sMessage% : %cUserName%';
cLogInfo        = 'Process:%cThisProcName% run with parameters pLogOutput=%pLogOutput%, pInputString=%pInputString%, pUndesirableFileSystem=%pUndesirableFileSystem%, pUndesirable1st=%pUndesirable1st%, pChanges=%pChanges%, pReplaceIfNotFound=%pReplaceIfNotFound%, pDelim=%pDelim%, pSeperator=%pSeperator%, pMode=%pMode%';

# Variables
nErrors         = 0;
#EndRegion

#Region # LogOutput parameters
IF( pLogoutput = 1 );
    LogOutput('INFO', Expand( cLogInfo ) );
ENDIF;
#EndRegion

#Region # Validate parameters
## Validate pInputString parameter
IF( Trim( pInputString ) @= '' );
    nErrors     =1;
    sMessage    = Expand('No element name specified in pInputString.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ELSE;
    sElementToUpdate        = Trim( pInputString ) ;
ENDIF;

## Validate pMode parameter
IF( pMode <>1 & pMode <>2 & pMode <>3 );
    nErrors     =1;
    sMessage    = Expand('pMode parameter must be 1, 2 or 3 not %pMode%.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ENDIF;

## Validate pDelim parameter
IF( Trim( pChanges ) @<> '' );
IF( Trim( pDelim ) @= '' );
    nErrors     = 1;
    sMessage    = Expand('No delimiter specified in pDelim.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ELSE;
    sDelim      = SUBST( Trim( pDelim ) , 1 , 1 );
ENDIF;
ENDIF;

## Validate pSeperator parameter
IF( Trim( pChanges ) @<> '' );
IF( Trim( pSeperator ) @= '' );
    nErrors     = 1;
    sMessage    = Expand('No seperator specified in pSeperator.');
    LogOutput( cMsgErrorLevel, Expand( cMsgErrorContent ) );
ELSE;
    sSeperator      = SUBST( Trim( pSeperator ) , 1 , 1 );
ENDIF;
ENDIF;

## Validate pChanges parameter
#pChanges        = Trim( pChanges );
IF( pChanges @= '' );
    
ELSEIF( SUBST( pChanges , LONG( pChanges ) , 1 )@<> sDelim );
    pChanges    = pChanges | sDelim ;
ENDIF;

#pChanges        = Trim( pChanges );
IF( pReplaceIfNotFound @= '' );
    sReplaceIfNotFound  = '';
ELSE;
    sReplaceIfNotFound  = Trim( pReplaceIfNotFound );
ENDIF;

##### Check for errors before continuing
If( nErrors <> 0 );
  If( pStrictErrorHandling = 1 ); 
      ProcessQuit; 
  Else;
      ProcessBreak;
  EndIf;
EndIf;
#EndRegion 

#Region # Prepare for While loop to validate each character seperately
sEle                        = TRIM( pInputString );
nEle                        = LONG( sEle );
sOutputString               = '';
nCount                      = 1;
# Loop through each character to see if valid
# If no script inlcuded in pChanges then the invalid character will be replaced  with pReplaceIfNotFound
WHILE( nCount <= nEle );
    sChar                   = SUBST( sEle , nCount , 1 );
    sChanges                = TRIM( pChanges );
    nUndesirableFileSystem  = SCAN( sChar , pUndesirableFileSystem );
    nUndesirable1st         = SCAN( sChar , pUndesirable1st );
    ## Test if sChar contains undesirable 
    IF( nUndesirableFileSystem >0 & ( pMode=1 % pMode=3) );
        ## Test if sChar in pChanges
        nChange             = SCAN( sChar , sChanges );
        IF( nChange >0 );
            sChanges        = SUBST( sChanges , nChange , 999 );
            nNewLong        = SCAN( sDelim , sChanges );       
            sNew            = SUBST( sChanges , 3  , nNewLong-3 );
            #sOutputString   = sOutputString | sNew ;
        ELSE;
            sNew            = sReplaceIfNotFound ;
        ENDIF;   
    ELSEIF( nUndesirable1st >0 & nCount=1 & ( pMode=2 % pMode=3) );
        ## Test if sChar in pChanges
        nChange             = SCAN( sChar , sChanges );
        IF( nChange >0 );
            sChanges        = SUBST( sChanges , nChange , 999 );
            nNewLong        = SCAN( sDelim , sChanges );       
            sNew            = SUBST( sChanges , 3  , nNewLong-3 );
            #sOutputString   = sOutputString | sNew ;
        ELSE;
            sNew            = pReplaceIfNotFound ;
        ENDIF;        
    ELSE;
        sNew                = sChar;
    ENDIF;
    sOutputString           = sOutputString | sNew ;
    # Loop through the rest of the characters
    nCount                  = nCount + 1 ;
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
##~~Join the bedrock TM1 community on GitHub https://github.com/cubewise-code/bedrock Ver 4.0.0~~##
################################################################################################# 

IF( nCount <> 1000000 );
    #AttrPutS( NumberToString( StringToNumber( sUpdatesNew ) )  , sDim2 , sUpdatesNew , 'Number');
ENDIF;

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
    sProcessAction = Expand( 'Process:%cThisProcName% has validated the string the element "%pInputString%" and returned "%sOutputString%".' );
    sProcessReturnCode = Expand( '%sProcessReturnCode% %sProcessAction%' );
    nProcessReturnCode = 1;
    If( pLogoutput = 1 );
        LogOutput('INFO', Expand( sProcessAction ) );   
    EndIf;
EndIf;

### End Epilog ###
#endregion