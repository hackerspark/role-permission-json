<script>
    import XLSX from "xlsx";
  
    const SHEET_NAME = "Roles-Permission-Access";
  
    const FEATURE_NAMES_ROW_INDEX = 0;
    const FEATURE_NAMES_COLUMN_INDEX = 3;
  
    const ACTION_NAMES_ROW_INDEX = 1;
  
    const DATA_ROW_INDEX = 2;
  
    async function processFile({ target: { files } }) {
      const sheetJSON = await getSheetJSON(files[0], SHEET_NAME);
  
      const TOTAl_ROW_COUNT = sheetJSON.length;
  
      const FEATURE_NAMES = sheetJSON[FEATURE_NAMES_ROW_INDEX].slice(
        FEATURE_NAMES_COLUMN_INDEX
      );
  
      const featureRangeDetails = getFeatureRangeDetails(
        FEATURE_NAMES,
        FEATURE_NAMES_COLUMN_INDEX
      );
  
      const finalMapping = generateMapping(
        sheetJSON,
        featureRangeDetails,
        FEATURE_NAMES_COLUMN_INDEX,
        DATA_ROW_INDEX,
        TOTAl_ROW_COUNT
      );
    }
  
    function generateMapping(
      sheetJSON,
      featureRangeDetails,
      FEATURE_NAMES_COLUMN_INDEX,
      DATA_ROW_INDEX,
      TOTAl_ROW_COUNT
    ) {
      let mapping = {};
  
      for (let index = DATA_ROW_INDEX; index < TOTAl_ROW_COUNT; index++) {
        const roleDetails = getRoleDetails(
          sheetJSON,
          index,
          FEATURE_NAMES_COLUMN_INDEX
        );
  
        let target = mapping;
        roleDetails.forEach(roleDetail => {
          if (!target[roleDetail]) {
            target[roleDetail] = {};
          }
          target = target[roleDetail];
        });
  
        let innerTarget = target;
        featureRangeDetails.forEach(featureDetail => {
          if (!target[featureDetail.featureName]) {
            target[featureDetail.featureName] = {};
          }
  
          const actionDetails = getActionDetails(
            sheetJSON,
            ACTION_NAMES_ROW_INDEX,
            featureDetail
          );
          actionDetails.forEach((action, i) => {
            target[featureDetail.featureName][action] = convertPermissionToBool(
              sheetJSON[index][featureDetail.startIndex + i]
            );
          });
        });
      }
      console.log(mapping);
    }
  
    function getActionDetails(sheetJSON, ACTION_NAMES_ROW_INDEX, featureDetail) {
      const targetRow = sheetJSON[ACTION_NAMES_ROW_INDEX];
      return targetRow.slice(featureDetail.startIndex, featureDetail.endIndex + 1);
    }
  
    function getRoleDetails(sheetJSON, index, FEATURE_NAMES_COLUMN_INDEX) {
      const targetRow = sheetJSON[index];
  
      let roleDetail = [];
      for (let colIndex = 0; colIndex < FEATURE_NAMES_COLUMN_INDEX; colIndex++) {
        roleDetail.push(getNonEmptyColValue(sheetJSON, index, colIndex));
      }
      return roleDetail;
    }
  
    function getNonEmptyColValue(sheetJSON, rowIndex, colIndex) {
      let targetValue = "";
      while (rowIndex > 0) {
        targetValue = sheetJSON[rowIndex][colIndex] || targetValue;
        rowIndex--;
        if (targetValue) {
          break;
        }
      }
      return targetValue;
    }
  
    async function getSheetJSON(file, sheetName) {
      const data = await file.arrayBuffer();
  
      const workbook = XLSX.read(data);
      const targetSheet = workbook.Sheets[sheetName];
  
      const sheetJSON = XLSX.utils.sheet_to_json(targetSheet, {
        raw: true,
        header: 1
      });
  
      return sheetJSON;
    }
  
    function getFeatureRangeDetails(FEATURE_NAMES, FEATURE_NAMES_COLUMN_INDEX) {
      let rangeDetails = [];
      for (let index = 0; index < FEATURE_NAMES.length; index++) {
        let featureName = FEATURE_NAMES[index];
        if (featureName) {
          if (rangeDetails[rangeDetails.length - 1]) {
            rangeDetails[rangeDetails.length - 1].endIndex =
              FEATURE_NAMES_COLUMN_INDEX + index;
          }
          rangeDetails.push({
            featureName,
            startIndex: FEATURE_NAMES_COLUMN_INDEX + index
          });
        }
      }
      return rangeDetails;
    }
  
    function convertPermissionToBool(permission) {
      let parsedPermission = permission || "";
      parsedPermission = parsedPermission.toLowerCase();
  
      return parsedPermission === "accessible";
    }
  </script>
  
  <style>
  </style>
  
  <main>
      <label for="file">Input File</label>
      <input type="file" id="file" on:change={processFile}/>
  </main>